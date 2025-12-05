# Static House Rental System
## Rentable Town Buildings for 51alpha

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **Authors**: 51alpha Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **Dependencies**: BaseHouse system, Region system
- **PR Ready**: No (Conceptual Design)

---

## 1. Executive Summary

The Static House Rental System allows GMs to create rentable regions around existing static town buildings. Players can rent these buildings like regular houses, gaining access to storage, lockdowns, co-tenants, and all standard house features.

### Core Concept

**Problem**: Static town buildings (taverns, shops, apartments) are visual-only with no house functionality.

**Solution**: Overlay an invisible `BaseHouse` multi on top of static buildings, giving them full house capabilities while preserving the original static appearance.

### Key Features

| Feature | Description |
|---------|-------------|
| **Invisible Multi** | House region without visible foundation |
| **Full House Features** | Lockdowns, secures, co-owners, bans |
| **Rent System** | Weekly payments from bank |
| **Auto-Eviction** | Clears storage on non-payment |
| **GM Management** | Complete Gump-based admin tools |
| **Player Familiar UI** | Looks like standard house sign |

---

## 2. Architecture Overview

### 2.1 Why Extend BaseHouse?

By inheriting from `BaseHouse`, we get all of this for free:

- ✅ House lockdowns
- ✅ Secure containers
- ✅ House regions (teleport blocking, etc.)
- ✅ House banning
- ✅ Friend/Co-Owner systems
- ✅ House sign interactions
- ✅ Decay timers (repurposed for rent)
- ✅ Storage rules
- ✅ Door access control

### 2.2 System Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Static Building                           │
│                  (Visual from map file)                      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   RentableBuilding                           │
│                (Invisible BaseHouse multi)                   │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Region    │  │   Storage   │  │   Access Control    │  │
│  │  Boundary   │  │  (Secures,  │  │  (Owner, Friends,   │  │
│  │             │  │  Lockdowns) │  │   Co-Tenants, Bans) │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  RentableBuildingSign                        │
│              (Visible sign for interaction)                  │
│                                                              │
│  Double-click → Player Gump (rent, manage, info)            │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Data Model

### 3.1 RentableBuilding Class

```csharp
public class RentableBuilding : BaseHouse
{
    // ═══ RENTAL PROPERTIES ═══
    
    [CommandProperty(AccessLevel.GameMaster)]
    public string BuildingName { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public int WeeklyRentPrice { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public TimeSpan RentalDuration { get; set; }  // How long each rental period lasts
    
    [CommandProperty(AccessLevel.GameMaster)]
    public Mobile Renter { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public DateTime RentalStartDate { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public DateTime NextPaymentDue { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public DateTime RentalExpires { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public RentalStatus Status { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public bool IsEnabled { get; set; }  // GM can disable rentals
    
    // ═══ STORAGE LIMITS ═══
    
    [CommandProperty(AccessLevel.GameMaster)]
    public int MaxLockdowns { get; set; }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public int MaxSecures { get; set; }
    
    // ═══ PAYMENT TRACKING ═══
    
    public int TotalRentCollected { get; set; }
    public int MissedPayments { get; set; }
    public DateTime LastPaymentDate { get; set; }
    
    // ═══ CO-TENANTS ═══
    
    public List<Mobile> CoTenants { get; set; } = new();
    public const int MaxCoTenants = 5;
    
    // ═══ GRACE PERIOD ═══
    
    public static readonly TimeSpan GracePeriod = TimeSpan.FromHours(48);
    public static readonly TimeSpan StorageClearDelay = TimeSpan.FromHours(24);
}

public enum RentalStatus
{
    Available,      // No renter, available for rent
    Occupied,       // Currently rented
    PaymentDue,     // Payment overdue, in grace period
    Evicting,       // Grace period expired, storage being cleared
    Disabled        // GM disabled this rental
}
```

### 3.2 Database Schema

```sql
-- Rental building tracking
CREATE TABLE s51a_rental_buildings (
    building_id SERIAL PRIMARY KEY,
    house_serial BIGINT NOT NULL UNIQUE,  -- ModernUO BaseHouse.Serial
    building_name VARCHAR(100) NOT NULL,
    
    -- Location
    location_x INT NOT NULL,
    location_y INT NOT NULL,
    location_z INT NOT NULL,
    map_id INT DEFAULT 0,
    
    -- Region bounds
    region_x1 INT NOT NULL,
    region_y1 INT NOT NULL,
    region_x2 INT NOT NULL,
    region_y2 INT NOT NULL,
    
    -- Rental settings
    weekly_rent_price INT NOT NULL DEFAULT 10000,
    rental_duration_days INT NOT NULL DEFAULT 7,
    max_lockdowns INT NOT NULL DEFAULT 500,
    max_secures INT NOT NULL DEFAULT 10,
    is_enabled BOOLEAN DEFAULT TRUE,
    
    -- Current rental
    renter_serial BIGINT,
    renter_account VARCHAR(100),
    rental_start_date TIMESTAMP WITH TIME ZONE,
    rental_expires TIMESTAMP WITH TIME ZONE,
    next_payment_due TIMESTAMP WITH TIME ZONE,
    status VARCHAR(20) DEFAULT 'Available',
    
    -- Stats
    total_rent_collected BIGINT DEFAULT 0,
    times_rented INT DEFAULT 0,
    
    -- Metadata
    created_by VARCHAR(100),  -- GM who created it
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_rental_buildings_location ON s51a_rental_buildings(location_x, location_y, map_id);
CREATE INDEX idx_rental_buildings_renter ON s51a_rental_buildings(renter_serial) WHERE renter_serial IS NOT NULL;
CREATE INDEX idx_rental_buildings_status ON s51a_rental_buildings(status);

-- Rental payment history
CREATE TABLE s51a_rental_payments (
    payment_id BIGSERIAL PRIMARY KEY,
    building_id INT NOT NULL REFERENCES s51a_rental_buildings(building_id),
    renter_serial BIGINT NOT NULL,
    amount INT NOT NULL,
    payment_type VARCHAR(20) NOT NULL,  -- 'initial', 'renewal', 'auto'
    paid_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_rental_payments_building ON s51a_rental_payments(building_id, paid_at DESC);

-- Co-tenant tracking
CREATE TABLE s51a_rental_cotenants (
    building_id INT NOT NULL REFERENCES s51a_rental_buildings(building_id),
    cotenant_serial BIGINT NOT NULL,
    added_by_serial BIGINT NOT NULL,
    added_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    PRIMARY KEY (building_id, cotenant_serial)
);

-- Eviction log
CREATE TABLE s51a_rental_evictions (
    eviction_id BIGSERIAL PRIMARY KEY,
    building_id INT NOT NULL,
    renter_serial BIGINT NOT NULL,
    renter_account VARCHAR(100),
    reason VARCHAR(50) NOT NULL,  -- 'non_payment', 'gm_eviction', 'voluntary'
    items_cleared INT DEFAULT 0,
    evicted_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 4. GM Rental Manager System

### 4.1 Main Command

```csharp
[Usage("[RentBuilding")]
[Description("Opens the Rentable Building Manager")]
public static void RentBuilding_OnCommand(CommandEventArgs e)
{
    e.Mobile.SendGump(new RentalManagerGump(e.Mobile as PlayerMobile));
}
```

### 4.2 GM Manager Gump

```
╔══════════════════════════════════════════════════════════════╗
║           RENTABLE BUILDING MANAGER                          ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  ┌──────────────────────────────────────────────────────┐   ║
║  │  [ 1 ] Create Rental Building                        │   ║
║  │        Target a location to place invisible multi    │   ║
║  └──────────────────────────────────────────────────────┘   ║
║                                                              ║
║  ┌──────────────────────────────────────────────────────┐   ║
║  │  [ 2 ] Select Existing Rental Building               │   ║
║  │        Target a rental sign to manage                │   ║
║  └──────────────────────────────────────────────────────┘   ║
║                                                              ║
║  ═══════════════ SELECTED BUILDING ════════════════════════ ║
║                                                              ║
║  Building: [None Selected]                                   ║
║  Status:   N/A                                               ║
║  Renter:   N/A                                               ║
║                                                              ║
║  ┌─────────────────┐  ┌─────────────────┐                   ║
║  │ [ 3 ] Set Rent  │  │ [ 4 ] Set       │                   ║
║  │       Price     │  │       Duration  │                   ║
║  └─────────────────┘  └─────────────────┘                   ║
║                                                              ║
║  ┌─────────────────┐  ┌─────────────────┐                   ║
║  │ [ 5 ] Set       │  │ [ 6 ] Set       │                   ║
║  │       Lockdowns │  │       Secures   │                   ║
║  └─────────────────┘  └─────────────────┘                   ║
║                                                              ║
║  ┌─────────────────┐  ┌─────────────────┐                   ║
║  │ [ 7 ] Assign    │  │ [ 8 ] Evict     │                   ║
║  │       Renter    │  │       Renter    │                   ║
║  └─────────────────┘  └─────────────────┘                   ║
║                                                              ║
║  ┌─────────────────┐  ┌─────────────────┐                   ║
║  │ [ 9 ] Toggle    │  │ [10 ] Delete    │                   ║
║  │       Enabled   │  │       Rental    │                   ║
║  └─────────────────┘  └─────────────────┘                   ║
║                                                              ║
║  ┌──────────────────────────────────────────────────────┐   ║
║  │  [11 ] README / HOW-TO GUIDE                         │   ║
║  └──────────────────────────────────────────────────────┘   ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### 4.3 GM Manager Implementation

```csharp
public class RentalManagerGump : Gump
{
    private PlayerMobile _gm;
    private RentableBuilding _selectedBuilding;
    
    public RentalManagerGump(PlayerMobile gm, RentableBuilding selected = null) : base(50, 50)
    {
        _gm = gm;
        _selectedBuilding = selected;
        
        Closable = true;
        Disposable = true;
        Draggable = true;
        
        AddBackground(0, 0, 450, 550, 9200);
        AddLabel(130, 20, 0x35, "RENTABLE BUILDING MANAGER");
        
        // Create button
        AddButton(20, 60, 4005, 4007, 1, GumpButtonType.Reply, 0);
        AddLabel(55, 60, 0, "Create Rental Building");
        AddLabel(55, 80, 0x3B2, "Target a location to place invisible multi");
        
        // Select button
        AddButton(20, 110, 4005, 4007, 2, GumpButtonType.Reply, 0);
        AddLabel(55, 110, 0, "Select Existing Rental Building");
        AddLabel(55, 130, 0x3B2, "Target a rental sign to manage");
        
        // Separator
        AddLabel(20, 165, 0x35, "═══════════ SELECTED BUILDING ════════════");
        
        // Selected building info
        if (_selectedBuilding != null)
        {
            AddLabel(20, 190, 0, $"Building: {_selectedBuilding.BuildingName}");
            AddLabel(20, 210, 0, $"Status:   {_selectedBuilding.Status}");
            AddLabel(20, 230, 0, $"Renter:   {_selectedBuilding.Renter?.Name ?? "None"}");
            AddLabel(20, 250, 0, $"Rent:     {_selectedBuilding.WeeklyRentPrice:N0} gold/week");
            AddLabel(20, 270, 0, $"Storage:  {_selectedBuilding.MaxLockdowns} lockdowns, {_selectedBuilding.MaxSecures} secures");
        }
        else
        {
            AddLabel(20, 190, 0x22, "No building selected");
            AddLabel(20, 210, 0x3B2, "Use 'Select Existing' or 'Create' first");
        }
        
        int y = 300;
        bool hasSelection = _selectedBuilding != null;
        int labelHue = hasSelection ? 0 : 0x3B2;
        
        // Row 1: Price and Duration
        AddButton(20, y, 4005, 4007, 3, GumpButtonType.Reply, 0);
        AddLabel(55, y, labelHue, "Set Rent Price");
        AddButton(220, y, 4005, 4007, 4, GumpButtonType.Reply, 0);
        AddLabel(255, y, labelHue, "Set Duration");
        
        y += 35;
        
        // Row 2: Lockdowns and Secures
        AddButton(20, y, 4005, 4007, 5, GumpButtonType.Reply, 0);
        AddLabel(55, y, labelHue, "Set Lockdowns");
        AddButton(220, y, 4005, 4007, 6, GumpButtonType.Reply, 0);
        AddLabel(255, y, labelHue, "Set Secures");
        
        y += 35;
        
        // Row 3: Assign and Evict
        AddButton(20, y, 4005, 4007, 7, GumpButtonType.Reply, 0);
        AddLabel(55, y, labelHue, "Assign Renter");
        AddButton(220, y, 4005, 4007, 8, GumpButtonType.Reply, 0);
        AddLabel(255, y, labelHue, "Evict Renter");
        
        y += 35;
        
        // Row 4: Toggle and Delete
        AddButton(20, y, 4005, 4007, 9, GumpButtonType.Reply, 0);
        AddLabel(55, y, labelHue, _selectedBuilding?.IsEnabled ?? true ? "Disable Rental" : "Enable Rental");
        AddButton(220, y, 4005, 4007, 10, GumpButtonType.Reply, 0);
        AddLabel(255, y, 0x22, "Delete Rental");
        
        y += 50;
        
        // README
        AddButton(20, y, 4005, 4007, 11, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0x35, "README / HOW-TO GUIDE");
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        switch (info.ButtonID)
        {
            case 1: // Create
                _gm.SendMessage("Target the ground where you want to place the rental building region.");
                _gm.Target = new CreateRentalBuildingTarget();
                break;
                
            case 2: // Select
                _gm.SendMessage("Target a rental building sign.");
                _gm.Target = new SelectRentalBuildingTarget(_gm);
                break;
                
            case 3: // Set Price
                if (_selectedBuilding == null) { NoSelection(); break; }
                _gm.SendMessage("Enter the weekly rent price in gold:");
                _gm.Prompt = new SetRentPricePrompt(_selectedBuilding, _gm);
                break;
                
            case 4: // Set Duration
                if (_selectedBuilding == null) { NoSelection(); break; }
                _gm.SendMessage("Enter the rental duration in days:");
                _gm.Prompt = new SetDurationPrompt(_selectedBuilding, _gm);
                break;
                
            case 5: // Set Lockdowns
                if (_selectedBuilding == null) { NoSelection(); break; }
                _gm.SendMessage("Enter the maximum lockdowns allowed:");
                _gm.Prompt = new SetLockdownsPrompt(_selectedBuilding, _gm);
                break;
                
            case 6: // Set Secures
                if (_selectedBuilding == null) { NoSelection(); break; }
                _gm.SendMessage("Enter the maximum secure containers allowed:");
                _gm.Prompt = new SetSecuresPrompt(_selectedBuilding, _gm);
                break;
                
            case 7: // Assign Renter
                if (_selectedBuilding == null) { NoSelection(); break; }
                _gm.SendMessage("Target the player to assign as renter.");
                _gm.Target = new AssignRenterTarget(_selectedBuilding, _gm);
                break;
                
            case 8: // Evict
                if (_selectedBuilding == null) { NoSelection(); break; }
                if (_selectedBuilding.Renter == null)
                {
                    _gm.SendMessage(0x22, "This building has no renter to evict.");
                    break;
                }
                _gm.SendGump(new ConfirmEvictionGump(_selectedBuilding, _gm));
                break;
                
            case 9: // Toggle Enabled
                if (_selectedBuilding == null) { NoSelection(); break; }
                _selectedBuilding.IsEnabled = !_selectedBuilding.IsEnabled;
                _selectedBuilding.Status = _selectedBuilding.IsEnabled ? RentalStatus.Available : RentalStatus.Disabled;
                _gm.SendMessage(0x35, $"Rental is now {(_selectedBuilding.IsEnabled ? "ENABLED" : "DISABLED")}");
                _gm.SendGump(new RentalManagerGump(_gm, _selectedBuilding));
                break;
                
            case 10: // Delete
                if (_selectedBuilding == null) { NoSelection(); break; }
                _gm.SendGump(new ConfirmDeleteRentalGump(_selectedBuilding, _gm));
                break;
                
            case 11: // README
                _gm.SendGump(new RentalReadmeGump(_gm, _selectedBuilding));
                break;
        }
    }
    
    private void NoSelection()
    {
        _gm.SendMessage(0x22, "You must select a rental building first.");
        _gm.SendGump(new RentalManagerGump(_gm, null));
    }
}
```

### 4.4 README Gump (Multi-Page)

```csharp
public class RentalReadmeGump : Gump
{
    private int _page;
    private const int MaxPages = 4;
    
    public RentalReadmeGump(PlayerMobile gm, RentableBuilding selected, int page = 1) : base(50, 50)
    {
        _page = Math.Clamp(page, 1, MaxPages);
        
        AddBackground(0, 0, 500, 450, 9200);
        AddLabel(150, 20, 0x35, $"RENTAL SYSTEM README - Page {_page}/{MaxPages}");
        
        switch (_page)
        {
            case 1: AddPage1(); break;
            case 2: AddPage2(); break;
            case 3: AddPage3(); break;
            case 4: AddPage4(); break;
        }
        
        // Navigation
        if (_page > 1)
        {
            AddButton(20, 400, 4014, 4016, 1, GumpButtonType.Reply, 0);
            AddLabel(55, 400, 0, "Previous");
        }
        
        if (_page < MaxPages)
        {
            AddButton(400, 400, 4005, 4007, 2, GumpButtonType.Reply, 0);
            AddLabel(435, 400, 0, "Next");
        }
        
        AddButton(220, 400, 4017, 4019, 0, GumpButtonType.Reply, 0);
        AddLabel(255, 400, 0, "Close");
    }
    
    private void AddPage1()
    {
        // WHAT THIS SYSTEM DOES
        AddLabel(20, 50, 0x35, "WHAT THIS SYSTEM DOES");
        AddHtml(20, 75, 460, 300, @"
<p>The Rentable Building System allows GMs to create rentable regions around existing static town buildings.</p>

<p><b>How It Works:</b></p>
<p>• An invisible 'multi house' is placed over a static building</p>
<p>• This gives the static all the features of a real house</p>
<p>• Players rent it like a normal house</p>
<p>• The building visuals come from the map - we don't change them</p>

<p><b>Features:</b></p>
<p>• Full lockdown and secure container support</p>
<p>• Co-tenant system (like house friends)</p>
<p>• Automatic weekly rent from player's bank</p>
<p>• Auto-eviction on non-payment after grace period</p>
<p>• GM can set custom prices, storage limits, etc.</p>

<p><b>What Players See:</b></p>
<p>• A sign on the building they can double-click</p>
<p>• Standard house-like interface for renting/managing</p>
<p>• Their items protected inside the rental region</p>
        ", false, true);
    }
    
    private void AddPage2()
    {
        // HOW TO CREATE A RENTABLE BUILDING
        AddLabel(20, 50, 0x35, "HOW TO CREATE A RENTABLE BUILDING");
        AddHtml(20, 75, 460, 300, @"
<p><b>Step-by-Step Guide:</b></p>

<p>1. Walk to the static building you want to make rentable</p>
<p>2. Use the command: <b>[RentBuilding</b></p>
<p>3. Click <b>'Create Rental Building'</b></p>
<p>4. Target the ground inside/near the building</p>
<p>5. A dialog will ask for the region boundaries</p>
<p>6. Walk to each corner and target to define the area</p>
<p>7. The system places an invisible multi + sign</p>

<p><b>After Creation:</b></p>
<p>• Select the building using 'Select Existing'</p>
<p>• Set the rent price (gold per week)</p>
<p>• Set the rental duration (days)</p>
<p>• Set lockdown and secure limits</p>
<p>• Enable the rental</p>

<p><b>Sign Placement:</b></p>
<p>• The sign spawns at the target location</p>
<p>• You can move it using [move command</p>
<p>• Place it where players can easily find it</p>
        ", false, true);
    }
    
    private void AddPage3()
    {
        // PLAYER EXPERIENCE
        AddLabel(20, 50, 0x35, "PLAYER EXPERIENCE");
        AddHtml(20, 75, 460, 300, @"
<p><b>Renting a Building:</b></p>
<p>• Player double-clicks the rental sign</p>
<p>• Sees price, duration, storage limits</p>
<p>• Clicks 'Rent This Building'</p>
<p>• Gold is withdrawn from their bank</p>
<p>• They become the renter immediately</p>

<p><b>Managing Their Rental:</b></p>
<p>• Add/remove co-tenants (up to 5)</p>
<p>• Lock down items (up to limit)</p>
<p>• Place secure containers (up to limit)</p>
<p>• Renew rent before expiration</p>
<p>• Voluntarily abandon rental</p>

<p><b>Payment System:</b></p>
<p>• Rent is due weekly</p>
<p>• Auto-billed from bank at due date</p>
<p>• If insufficient funds: 48-hour grace period</p>
<p>• Warning messages sent at 24h and 12h</p>
<p>• After grace: eviction begins</p>

<p><b>Eviction:</b></p>
<p>• 24 hours to retrieve items after eviction notice</p>
<p>• After that, all items are cleared</p>
<p>• Building becomes available again</p>
        ", false, true);
    }
    
    private void AddPage4()
    {
        // TROUBLESHOOTING
        AddLabel(20, 50, 0x35, "TROUBLESHOOTING");
        AddHtml(20, 75, 460, 300, @"
<p><b>Region boundaries wrong?</b></p>
<p>• Delete the rental and recreate with correct bounds</p>
<p>• Or use [props on the building to adjust manually</p>

<p><b>Can't place rental?</b></p>
<p>• Check if another house/rental overlaps</p>
<p>• Check if the area is a protected region</p>
<p>• Make sure you're targeting valid ground</p>

<p><b>Sign missing/lost?</b></p>
<p>• Use [RentBuilding and 'Select Existing'</p>
<p>• Target the general area</p>
<p>• Or spawn a new sign linked to the building</p>

<p><b>Multi not working?</b></p>
<p>• Check server logs for errors</p>
<p>• Verify the BaseHouse was created properly</p>
<p>• Use [props to inspect the building object</p>

<p><b>Player can't access storage?</b></p>
<p>• Verify they are the renter or co-tenant</p>
<p>• Check if rental has expired</p>
<p>• Check if building is disabled</p>

<p><b>Rent not being collected?</b></p>
<p>• Check the rental timer is running</p>
<p>• Verify NextPaymentDue is set correctly</p>
<p>• Check server logs for payment errors</p>
        ", false, true);
    }
}
```

---

## 5. Player Rental Sign Gump

### 5.1 Sign Item

```csharp
public class RentableBuildingSign : Item
{
    private RentableBuilding _building;
    
    [Constructable]
    public RentableBuildingSign(RentableBuilding building) : base(0xBD2)  // House sign graphic
    {
        _building = building;
        Movable = false;
        Name = building.BuildingName ?? "Rental Building";
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (_building == null || _building.Deleted)
        {
            pm.SendMessage(0x22, "This rental building no longer exists.");
            return;
        }
        
        // GM gets GM gump
        if (pm.AccessLevel >= AccessLevel.GameMaster)
        {
            pm.SendGump(new RentalManagerGump(pm, _building));
            return;
        }
        
        // Player gets player gump
        pm.SendGump(new RentalSignGump(pm, _building));
    }
    
    public override void GetProperties(ObjectPropertyList list)
    {
        base.GetProperties(list);
        
        if (_building != null)
        {
            list.Add(1060658, $"Status\t{_building.Status}");  // ~1_val~: ~2_val~
            
            if (_building.Renter != null)
                list.Add(1060659, $"Renter\t{_building.Renter.Name}");
            else
                list.Add(1060659, $"Rent\t{_building.WeeklyRentPrice:N0} gold/week");
        }
    }
}
```

### 5.2 Player Sign Gump

```
╔══════════════════════════════════════════════════════════════╗
║         MARKET STREET FLAT #3                                ║
║              Rental Building                                 ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  ════════════════ RENTAL INFO ════════════════════════════  ║
║                                                              ║
║  Rent Price:      25,000 gold / week                        ║
║  Rental Duration: 4 weeks                                    ║
║                                                              ║
║  Storage Limits:                                             ║
║    • Lockdowns:   500 items                                  ║
║    • Secures:     10 containers                              ║
║                                                              ║
║  ════════════════ CURRENT STATUS ═════════════════════════  ║
║                                                              ║
║  Status:          AVAILABLE                                  ║
║                                                              ║
║  ════════════════════════════════════════════════════════   ║
║                                                              ║
║         ┌─────────────────────────────────┐                 ║
║         │     [ RENT THIS BUILDING ]      │                 ║
║         └─────────────────────────────────┘                 ║
║                                                              ║
║         ┌─────────────────────────────────┐                 ║
║         │     [ MORE INFORMATION ]        │                 ║
║         └─────────────────────────────────┘                 ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

**When Occupied (by current player):**

```
╔══════════════════════════════════════════════════════════════╗
║         MARKET STREET FLAT #3                                ║
║              Your Rental                                     ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  ════════════════ RENTAL INFO ════════════════════════════  ║
║                                                              ║
║  Rent Price:      25,000 gold / week                        ║
║  Time Remaining:  5 days, 12 hours                          ║
║  Next Payment:    December 10, 2025                         ║
║                                                              ║
║  Storage Used:                                               ║
║    • Lockdowns:   127 / 500                                  ║
║    • Secures:     3 / 10                                     ║
║                                                              ║
║  ════════════════ CO-TENANTS ═════════════════════════════  ║
║                                                              ║
║    1. PlayerTwo                                              ║
║    2. PlayerThree                                            ║
║    (3 slots available)                                       ║
║                                                              ║
║  ════════════════════════════════════════════════════════   ║
║                                                              ║
║  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       ║
║  │ RENEW RENT   │  │ ADD CO-      │  │ REMOVE CO-   │       ║
║  │              │  │ TENANT       │  │ TENANT       │       ║
║  └──────────────┘  └──────────────┘  └──────────────┘       ║
║                                                              ║
║  ┌──────────────┐  ┌──────────────┐                         ║
║  │ ABANDON      │  │ MORE INFO    │                         ║
║  │ RENTAL       │  │              │                         ║
║  └──────────────┘  └──────────────┘                         ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### 5.3 Player Gump Implementation

```csharp
public class RentalSignGump : Gump
{
    private PlayerMobile _player;
    private RentableBuilding _building;
    
    public RentalSignGump(PlayerMobile player, RentableBuilding building) : base(50, 50)
    {
        _player = player;
        _building = building;
        
        bool isRenter = building.Renter == player;
        bool isCoTenant = building.CoTenants.Contains(player);
        bool isAvailable = building.Status == RentalStatus.Available && building.IsEnabled;
        
        AddBackground(0, 0, 400, 480, 9200);
        
        // Title
        AddLabel(120, 20, 0x35, building.BuildingName ?? "Rental Building");
        AddLabel(150, 40, 0, isRenter ? "Your Rental" : "Rental Building");
        
        // Rental Info Section
        AddLabel(20, 70, 0x35, "════════════ RENTAL INFO ════════════");
        
        AddLabel(20, 95, 0, $"Rent Price: {building.WeeklyRentPrice:N0} gold / week");
        
        if (isRenter)
        {
            var remaining = building.RentalExpires - DateTime.UtcNow;
            AddLabel(20, 115, 0, $"Time Remaining: {FormatTimeSpan(remaining)}");
            AddLabel(20, 135, 0, $"Next Payment: {building.NextPaymentDue:MMM dd, yyyy}");
            
            // Storage used
            AddLabel(20, 165, 0, "Storage Used:");
            AddLabel(40, 185, 0, $"• Lockdowns: {building.LockDownCount} / {building.MaxLockdowns}");
            AddLabel(40, 205, 0, $"• Secures: {building.SecureCount} / {building.MaxSecures}");
        }
        else
        {
            var duration = TimeSpan.FromDays(building.RentalDuration.TotalDays);
            AddLabel(20, 115, 0, $"Rental Duration: {(int)duration.TotalDays} days");
            
            AddLabel(20, 145, 0, "Storage Limits:");
            AddLabel(40, 165, 0, $"• Lockdowns: {building.MaxLockdowns} items");
            AddLabel(40, 185, 0, $"• Secures: {building.MaxSecures} containers");
        }
        
        // Status Section
        int y = 230;
        AddLabel(20, y, 0x35, "════════════ STATUS ════════════════");
        y += 25;
        
        if (isRenter)
        {
            // Co-tenant list
            AddLabel(20, y, 0, "Co-Tenants:");
            y += 20;
            
            for (int i = 0; i < building.CoTenants.Count; i++)
            {
                AddLabel(40, y, 0, $"{i + 1}. {building.CoTenants[i].Name}");
                y += 18;
            }
            
            int slotsAvailable = RentableBuilding.MaxCoTenants - building.CoTenants.Count;
            AddLabel(40, y, 0x3B2, $"({slotsAvailable} slots available)");
            y += 30;
        }
        else if (building.Renter != null)
        {
            AddLabel(20, y, 0, $"Renter: {building.Renter.Name}");
            AddLabel(20, y + 20, 0x22, "This building is currently rented.");
            y += 50;
        }
        else
        {
            string statusText = building.IsEnabled ? "AVAILABLE" : "NOT AVAILABLE";
            int statusHue = building.IsEnabled ? 0x35 : 0x22;
            AddLabel(20, y, 0, "Status:");
            AddLabel(100, y, statusHue, statusText);
            y += 40;
        }
        
        // Action buttons
        y = 380;
        
        if (isRenter)
        {
            // Renter actions
            AddButton(20, y, 4005, 4007, 1, GumpButtonType.Reply, 0);
            AddLabel(55, y, 0, "Renew Rent");
            
            AddButton(140, y, 4005, 4007, 2, GumpButtonType.Reply, 0);
            AddLabel(175, y, 0, "Add Co-Tenant");
            
            AddButton(280, y, 4005, 4007, 3, GumpButtonType.Reply, 0);
            AddLabel(315, y, 0, "Remove");
            
            y += 35;
            
            AddButton(20, y, 4005, 4007, 4, GumpButtonType.Reply, 0);
            AddLabel(55, y, 0x22, "Abandon Rental");
            
            AddButton(180, y, 4005, 4007, 5, GumpButtonType.Reply, 0);
            AddLabel(215, y, 0, "More Info");
        }
        else if (isAvailable)
        {
            // Available actions
            AddButton(100, y, 4005, 4007, 6, GumpButtonType.Reply, 0);
            AddLabel(135, y, 0x35, "RENT THIS BUILDING");
            
            y += 35;
            
            AddButton(130, y, 4005, 4007, 5, GumpButtonType.Reply, 0);
            AddLabel(165, y, 0, "More Information");
        }
        else
        {
            // Not available
            AddButton(130, y, 4005, 4007, 5, GumpButtonType.Reply, 0);
            AddLabel(165, y, 0, "More Information");
        }
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        switch (info.ButtonID)
        {
            case 1: // Renew Rent
                RentalManager.TryRenewRent(_player, _building);
                break;
                
            case 2: // Add Co-Tenant
                _player.SendMessage("Target the player to add as co-tenant.");
                _player.Target = new AddCoTenantTarget(_building);
                break;
                
            case 3: // Remove Co-Tenant
                _player.SendGump(new RemoveCoTenantGump(_player, _building));
                break;
                
            case 4: // Abandon
                _player.SendGump(new ConfirmAbandonGump(_player, _building));
                break;
                
            case 5: // More Info
                _player.SendGump(new RentalHelpGump(_player, _building));
                break;
                
            case 6: // Rent Building
                RentalManager.TryRentBuilding(_player, _building);
                break;
        }
    }
    
    private string FormatTimeSpan(TimeSpan ts)
    {
        if (ts.TotalDays >= 1)
            return $"{(int)ts.TotalDays} days, {ts.Hours} hours";
        return $"{ts.Hours} hours, {ts.Minutes} minutes";
    }
}
```

---

## 6. Rental Manager Logic

### 6.1 Core Rental Operations

```csharp
public static class RentalManager
{
    public static void Initialize()
    {
        // Start rental payment timer
        Timer.DelayCall(TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1), ProcessRentalPayments);
        
        Console.WriteLine("[51alpha] Rental system initialized.");
    }
    
    public static bool TryRentBuilding(PlayerMobile player, RentableBuilding building)
    {
        if (building.Status != RentalStatus.Available || !building.IsEnabled)
        {
            player.SendMessage(0x22, "This building is not available for rent.");
            return false;
        }
        
        // Check if player already rents another building
        // (Optional: limit 1 rental per account)
        
        // Check bank gold
        int totalCost = building.WeeklyRentPrice;  // First week's rent
        
        if (!Banker.Withdraw(player, totalCost))
        {
            player.SendMessage(0x22, $"You need {totalCost:N0} gold in your bank to rent this building.");
            return false;
        }
        
        // Assign rental
        building.Renter = player;
        building.RentalStartDate = DateTime.UtcNow;
        building.NextPaymentDue = DateTime.UtcNow.Add(building.RentalDuration);
        building.RentalExpires = building.NextPaymentDue;
        building.Status = RentalStatus.Occupied;
        building.TotalRentCollected += totalCost;
        
        // Log payment
        LogPayment(building, player, totalCost, "initial");
        
        player.SendMessage(0x35, $"You have rented {building.BuildingName} for {totalCost:N0} gold!");
        player.SendMessage(0x35, $"Your next payment is due in {(int)building.RentalDuration.TotalDays} days.");
        
        return true;
    }
    
    public static bool TryRenewRent(PlayerMobile player, RentableBuilding building)
    {
        if (building.Renter != player)
        {
            player.SendMessage(0x22, "You are not the renter of this building.");
            return false;
        }
        
        int cost = building.WeeklyRentPrice;
        
        if (!Banker.Withdraw(player, cost))
        {
            player.SendMessage(0x22, $"You need {cost:N0} gold in your bank to renew.");
            return false;
        }
        
        building.RentalExpires = building.RentalExpires.Add(building.RentalDuration);
        building.NextPaymentDue = building.NextPaymentDue.Add(building.RentalDuration);
        building.Status = RentalStatus.Occupied;
        building.TotalRentCollected += cost;
        building.MissedPayments = 0;
        
        LogPayment(building, player, cost, "renewal");
        
        player.SendMessage(0x35, $"Rent renewed! Your rental now expires on {building.RentalExpires:MMM dd, yyyy}.");
        
        return true;
    }
    
    public static void ProcessRentalPayments()
    {
        var now = DateTime.UtcNow;
        
        foreach (var building in GetAllRentals())
        {
            if (building.Renter == null || building.Status == RentalStatus.Disabled)
                continue;
                
            // Check if payment is due
            if (now >= building.NextPaymentDue)
            {
                ProcessPaymentDue(building);
            }
            
            // Check grace period expiration
            if (building.Status == RentalStatus.PaymentDue)
            {
                var graceExpires = building.NextPaymentDue + RentableBuilding.GracePeriod;
                
                if (now >= graceExpires)
                {
                    BeginEviction(building, "non_payment");
                }
                else
                {
                    // Send warnings
                    var remaining = graceExpires - now;
                    
                    if (remaining <= TimeSpan.FromHours(12) && !building.HasSent12HourWarning)
                    {
                        SendRenterMessage(building, 0x22, 
                            "WARNING: Your rental payment is overdue! You have 12 hours to pay or be evicted!");
                        building.HasSent12HourWarning = true;
                    }
                    else if (remaining <= TimeSpan.FromHours(24) && !building.HasSent24HourWarning)
                    {
                        SendRenterMessage(building, 0x22,
                            "WARNING: Your rental payment is overdue! You have 24 hours to pay.");
                        building.HasSent24HourWarning = true;
                    }
                }
            }
            
            // Process eviction completion
            if (building.Status == RentalStatus.Evicting)
            {
                var clearTime = building.EvictionStarted + RentableBuilding.StorageClearDelay;
                
                if (now >= clearTime)
                {
                    CompleteEviction(building);
                }
            }
        }
    }
    
    private static void ProcessPaymentDue(RentableBuilding building)
    {
        var renter = building.Renter as PlayerMobile;
        
        if (renter == null)
            return;
            
        // Try auto-payment from bank
        if (Banker.Withdraw(renter, building.WeeklyRentPrice))
        {
            building.NextPaymentDue = building.NextPaymentDue.Add(building.RentalDuration);
            building.RentalExpires = building.NextPaymentDue;
            building.TotalRentCollected += building.WeeklyRentPrice;
            
            LogPayment(building, renter, building.WeeklyRentPrice, "auto");
            
            SendRenterMessage(building, 0x35,
                $"Rent payment of {building.WeeklyRentPrice:N0} gold automatically collected from your bank.");
        }
        else
        {
            // Enter grace period
            building.Status = RentalStatus.PaymentDue;
            building.MissedPayments++;
            building.HasSent24HourWarning = false;
            building.HasSent12HourWarning = false;
            
            SendRenterMessage(building, 0x22,
                $"RENT OVERDUE: You have {(int)RentableBuilding.GracePeriod.TotalHours} hours to pay " +
                $"{building.WeeklyRentPrice:N0} gold or be evicted!");
        }
    }
    
    private static void BeginEviction(RentableBuilding building, string reason)
    {
        building.Status = RentalStatus.Evicting;
        building.EvictionStarted = DateTime.UtcNow;
        
        SendRenterMessage(building, 0x22,
            $"EVICTION NOTICE: You are being evicted from {building.BuildingName}! " +
            $"You have {(int)RentableBuilding.StorageClearDelay.TotalHours} hours to retrieve your belongings!");
        
        // Log eviction start
        LogEviction(building, reason, 0);
    }
    
    private static void CompleteEviction(RentableBuilding building)
    {
        // Clear all storage
        int itemsCleared = ClearBuildingStorage(building);
        
        // Reset building
        var formerRenter = building.Renter;
        
        building.Renter = null;
        building.CoTenants.Clear();
        building.Status = RentalStatus.Available;
        building.RentalStartDate = DateTime.MinValue;
        building.NextPaymentDue = DateTime.MinValue;
        building.RentalExpires = DateTime.MinValue;
        
        // Update eviction log
        UpdateEvictionLog(building, formerRenter, itemsCleared);
        
        if (formerRenter != null)
        {
            formerRenter.SendMessage(0x22, 
                $"You have been evicted from {building.BuildingName}. {itemsCleared} items were cleared.");
        }
    }
    
    private static int ClearBuildingStorage(RentableBuilding building)
    {
        int count = 0;
        
        // Release all lockdowns
        foreach (var item in building.LockDowns.ToList())
        {
            building.Release(building.Renter, item);
            item.Delete();
            count++;
        }
        
        // Clear all secure containers
        foreach (var info in building.Secures.ToList())
        {
            if (info.Item is Container container)
            {
                count += container.TotalItems;
                container.Delete();
            }
        }
        
        return count;
    }
    
    public static void AbandonRental(PlayerMobile player, RentableBuilding building)
    {
        if (building.Renter != player)
            return;
            
        int itemsCleared = ClearBuildingStorage(building);
        
        building.Renter = null;
        building.CoTenants.Clear();
        building.Status = RentalStatus.Available;
        
        LogEviction(building, "voluntary", itemsCleared);
        
        player.SendMessage(0x35, $"You have abandoned your rental of {building.BuildingName}.");
    }
    
    public static void GMEvict(RentableBuilding building, Mobile gm)
    {
        if (building.Renter == null)
            return;
            
        int itemsCleared = ClearBuildingStorage(building);
        var formerRenter = building.Renter;
        
        building.Renter = null;
        building.CoTenants.Clear();
        building.Status = RentalStatus.Available;
        
        LogEviction(building, "gm_eviction", itemsCleared);
        
        if (formerRenter != null)
        {
            formerRenter.SendMessage(0x22, 
                $"A Game Master has evicted you from {building.BuildingName}.");
        }
        
        gm.SendMessage(0x35, $"Evicted {formerRenter?.Name ?? "renter"} from {building.BuildingName}. " +
            $"{itemsCleared} items cleared.");
    }
    
    // Helper methods
    private static void SendRenterMessage(RentableBuilding building, int hue, string message)
    {
        if (building.Renter is PlayerMobile pm && pm.NetState != null)
        {
            pm.SendMessage(hue, message);
        }
    }
    
    private static void LogPayment(RentableBuilding building, Mobile renter, int amount, string type)
    {
        // Database logging
        Database.ExecuteAsync(
            "INSERT INTO s51a_rental_payments (building_id, renter_serial, amount, payment_type) " +
            "VALUES (@bid, @renter, @amount, @type)",
            new NpgsqlParameter("@bid", GetBuildingId(building)),
            new NpgsqlParameter("@renter", renter.Serial.Value),
            new NpgsqlParameter("@amount", amount),
            new NpgsqlParameter("@type", type)
        );
    }
    
    private static void LogEviction(RentableBuilding building, string reason, int itemsCleared)
    {
        Database.ExecuteAsync(
            "INSERT INTO s51a_rental_evictions (building_id, renter_serial, renter_account, reason, items_cleared) " +
            "VALUES (@bid, @renter, @account, @reason, @items)",
            new NpgsqlParameter("@bid", GetBuildingId(building)),
            new NpgsqlParameter("@renter", building.Renter?.Serial.Value ?? 0),
            new NpgsqlParameter("@account", (building.Renter as PlayerMobile)?.Account?.Username ?? ""),
            new NpgsqlParameter("@reason", reason),
            new NpgsqlParameter("@items", itemsCleared)
        );
    }
}
```

---

## 7. Co-Tenant System

### 7.1 Co-Tenant Management

```csharp
public static class CoTenantManager
{
    public static bool AddCoTenant(RentableBuilding building, Mobile renter, Mobile cotenant)
    {
        if (building.Renter != renter)
        {
            renter.SendMessage(0x22, "Only the renter can add co-tenants.");
            return false;
        }
        
        if (building.CoTenants.Count >= RentableBuilding.MaxCoTenants)
        {
            renter.SendMessage(0x22, $"Maximum of {RentableBuilding.MaxCoTenants} co-tenants allowed.");
            return false;
        }
        
        if (building.CoTenants.Contains(cotenant))
        {
            renter.SendMessage(0x22, "That player is already a co-tenant.");
            return false;
        }
        
        if (cotenant == renter)
        {
            renter.SendMessage(0x22, "You cannot add yourself as a co-tenant.");
            return false;
        }
        
        building.CoTenants.Add(cotenant);
        
        // Also add as house friend for access
        building.AddFriend(renter, cotenant);
        
        renter.SendMessage(0x35, $"{cotenant.Name} is now a co-tenant.");
        cotenant.SendMessage(0x35, $"You are now a co-tenant at {building.BuildingName}.");
        
        // Log to database
        Database.ExecuteAsync(
            "INSERT INTO s51a_rental_cotenants (building_id, cotenant_serial, added_by_serial) " +
            "VALUES (@bid, @cotenant, @addedby)",
            new NpgsqlParameter("@bid", GetBuildingId(building)),
            new NpgsqlParameter("@cotenant", cotenant.Serial.Value),
            new NpgsqlParameter("@addedby", renter.Serial.Value)
        );
        
        return true;
    }
    
    public static bool RemoveCoTenant(RentableBuilding building, Mobile renter, Mobile cotenant)
    {
        if (building.Renter != renter)
        {
            renter.SendMessage(0x22, "Only the renter can remove co-tenants.");
            return false;
        }
        
        if (!building.CoTenants.Contains(cotenant))
        {
            renter.SendMessage(0x22, "That player is not a co-tenant.");
            return false;
        }
        
        building.CoTenants.Remove(cotenant);
        building.RemoveFriend(renter, cotenant);
        
        renter.SendMessage(0x35, $"{cotenant.Name} is no longer a co-tenant.");
        cotenant.SendMessage(0x22, $"You are no longer a co-tenant at {building.BuildingName}.");
        
        // Remove from database
        Database.ExecuteAsync(
            "DELETE FROM s51a_rental_cotenants WHERE building_id = @bid AND cotenant_serial = @cotenant",
            new NpgsqlParameter("@bid", GetBuildingId(building)),
            new NpgsqlParameter("@cotenant", cotenant.Serial.Value)
        );
        
        return true;
    }
}
```

---

## 8. Building Creation

### 8.1 Create Rental Building Target

```csharp
public class CreateRentalBuildingTarget : Target
{
    public CreateRentalBuildingTarget() : base(-1, true, TargetFlags.None)
    {
    }
    
    protected override void OnTarget(Mobile from, object targeted)
    {
        if (!(from is PlayerMobile gm))
            return;
            
        IPoint3D p = targeted as IPoint3D;
        
        if (p == null)
        {
            gm.SendMessage(0x22, "Invalid target.");
            return;
        }
        
        Point3D location = new Point3D(p);
        
        // Open region definition gump
        gm.SendGump(new DefineRegionGump(gm, location));
    }
}

public class DefineRegionGump : Gump
{
    private Point3D _startLocation;
    private PlayerMobile _gm;
    
    public DefineRegionGump(PlayerMobile gm, Point3D start) : base(50, 50)
    {
        _gm = gm;
        _startLocation = start;
        
        AddBackground(0, 0, 350, 300, 9200);
        AddLabel(80, 20, 0x35, "DEFINE RENTAL REGION");
        
        AddLabel(20, 60, 0, $"Start Point: {start.X}, {start.Y}, {start.Z}");
        
        AddLabel(20, 100, 0, "Now target the opposite corner of the region.");
        AddLabel(20, 120, 0, "This defines the rectangular boundary.");
        
        AddLabel(20, 160, 0, "Building Name:");
        AddBackground(20, 180, 300, 25, 9300);
        AddTextEntry(25, 183, 290, 20, 0, 0, "New Rental Building");
        
        AddButton(80, 230, 4005, 4007, 1, GumpButtonType.Reply, 0);
        AddLabel(115, 230, 0x35, "Target Opposite Corner");
        
        AddButton(80, 260, 4017, 4019, 0, GumpButtonType.Reply, 0);
        AddLabel(115, 260, 0, "Cancel");
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        if (info.ButtonID != 1)
            return;
            
        string buildingName = info.GetTextEntry(0)?.Text ?? "Rental Building";
        
        _gm.SendMessage("Target the opposite corner of the region.");
        _gm.Target = new DefineRegionCornerTarget(_gm, _startLocation, buildingName);
    }
}

public class DefineRegionCornerTarget : Target
{
    private PlayerMobile _gm;
    private Point3D _start;
    private string _buildingName;
    
    public DefineRegionCornerTarget(PlayerMobile gm, Point3D start, string name) 
        : base(-1, true, TargetFlags.None)
    {
        _gm = gm;
        _start = start;
        _buildingName = name;
    }
    
    protected override void OnTarget(Mobile from, object targeted)
    {
        IPoint3D p = targeted as IPoint3D;
        
        if (p == null)
        {
            _gm.SendMessage(0x22, "Invalid target.");
            return;
        }
        
        Point3D end = new Point3D(p);
        
        // Calculate region bounds
        int x1 = Math.Min(_start.X, end.X);
        int y1 = Math.Min(_start.Y, end.Y);
        int x2 = Math.Max(_start.X, end.X);
        int y2 = Math.Max(_start.Y, end.Y);
        
        // Create the rental building
        var building = new RentableBuilding(_gm, _gm.Map, 
            new Rectangle2D(x1, y1, x2 - x1, y2 - y1));
        
        building.BuildingName = _buildingName;
        building.WeeklyRentPrice = 10000;  // Default
        building.RentalDuration = TimeSpan.FromDays(7);  // Default
        building.MaxLockdowns = 500;  // Default
        building.MaxSecures = 10;  // Default
        building.IsEnabled = false;  // Disabled until GM enables
        building.Status = RentalStatus.Disabled;
        
        // Place sign at start location
        var sign = new RentableBuildingSign(building);
        sign.MoveToWorld(_start, _gm.Map);
        building.Sign = sign;
        
        _gm.SendMessage(0x35, $"Rental building '{_buildingName}' created!");
        _gm.SendMessage(0x35, $"Region: ({x1}, {y1}) to ({x2}, {y2})");
        _gm.SendMessage(0x35, "Use [RentBuilding to configure and enable.");
        
        // Open manager with new building selected
        _gm.SendGump(new RentalManagerGump(_gm, building));
    }
}
```

---

## 9. Configuration

### 9.1 Config Values

```sql
-- Add to s51a_config
INSERT INTO s51a_config (config_key, config_value, value_type, description) VALUES
('rental.default_price', '10000', 'int', 'Default weekly rent price'),
('rental.default_duration_days', '7', 'int', 'Default rental duration in days'),
('rental.default_lockdowns', '500', 'int', 'Default lockdown limit'),
('rental.default_secures', '10', 'int', 'Default secure container limit'),
('rental.grace_period_hours', '48', 'int', 'Hours before eviction after missed payment'),
('rental.storage_clear_delay_hours', '24', 'int', 'Hours to retrieve items after eviction notice'),
('rental.max_cotenants', '5', 'int', 'Maximum co-tenants per rental'),
('rental.max_rentals_per_account', '1', 'int', 'Maximum rentals per account (0 = unlimited)');
```

---

## 10. Admin Commands

```csharp
public static class RentalCommands
{
    public static void Register()
    {
        CommandSystem.Register("RentBuilding", AccessLevel.GameMaster, RentBuilding_OnCommand);
        CommandSystem.Register("ListRentals", AccessLevel.GameMaster, ListRentals_OnCommand);
        CommandSystem.Register("RentalStats", AccessLevel.GameMaster, RentalStats_OnCommand);
    }
    
    [Usage("[RentBuilding")]
    [Description("Opens the Rental Building Manager")]
    private static void RentBuilding_OnCommand(CommandEventArgs e)
    {
        e.Mobile.SendGump(new RentalManagerGump(e.Mobile as PlayerMobile));
    }
    
    [Usage("[ListRentals")]
    [Description("Lists all rental buildings")]
    private static void ListRentals_OnCommand(CommandEventArgs e)
    {
        var rentals = RentalManager.GetAllRentals();
        
        e.Mobile.SendMessage(0x35, $"═══ Rental Buildings ({rentals.Count}) ═══");
        
        foreach (var r in rentals)
        {
            string status = r.Renter != null ? $"Rented by {r.Renter.Name}" : "Available";
            e.Mobile.SendMessage(0, $"  {r.BuildingName}: {status}");
        }
    }
    
    [Usage("[RentalStats")]
    [Description("Shows rental system statistics")]
    private static void RentalStats_OnCommand(CommandEventArgs e)
    {
        var rentals = RentalManager.GetAllRentals();
        int occupied = rentals.Count(r => r.Renter != null);
        long totalRent = rentals.Sum(r => r.TotalRentCollected);
        
        e.Mobile.SendMessage(0x35, "═══ Rental System Stats ═══");
        e.Mobile.SendMessage(0, $"  Total Buildings: {rentals.Count}");
        e.Mobile.SendMessage(0, $"  Occupied: {occupied}");
        e.Mobile.SendMessage(0, $"  Available: {rentals.Count - occupied}");
        e.Mobile.SendMessage(0, $"  Total Rent Collected: {totalRent:N0} gold");
    }
}
```

---

## 11. Testing Checklist

### GM Functions
- [ ] `[RentBuilding` opens manager gump
- [ ] Can create new rental building
- [ ] Region boundaries set correctly
- [ ] Can select existing building
- [ ] Can set rent price
- [ ] Can set duration
- [ ] Can set lockdown limit
- [ ] Can set secure limit
- [ ] Can assign renter manually
- [ ] Can evict renter
- [ ] Can toggle enabled/disabled
- [ ] Can delete rental
- [ ] README pages display correctly

### Player Functions
- [ ] Sign double-click opens gump
- [ ] Available buildings show rent button
- [ ] Can rent building (gold withdrawn)
- [ ] Can renew rent
- [ ] Can add co-tenant
- [ ] Can remove co-tenant
- [ ] Can abandon rental
- [ ] Storage (lockdowns/secures) works
- [ ] Co-tenants have access

### Payment System
- [ ] Auto-payment from bank works
- [ ] Grace period triggers on failed payment
- [ ] Warning messages sent at 24h and 12h
- [ ] Eviction begins after grace period
- [ ] Storage cleared after delay
- [ ] Building becomes available after eviction

### Edge Cases
- [ ] Server restart preserves rental state
- [ ] Renter logs out - state preserved
- [ ] Renter deletes character - eviction?
- [ ] GM disables building during active rental
- [ ] Multiple GMs editing same building

---

## 12. Implementation Files

| File | Purpose |
|------|---------|
| `Sphere51a/Rental/RentableBuilding.cs` | Main building class extending BaseHouse |
| `Sphere51a/Rental/RentableBuildingSign.cs` | Sign item for interaction |
| `Sphere51a/Rental/RentalManager.cs` | Core rental operations |
| `Sphere51a/Rental/CoTenantManager.cs` | Co-tenant management |
| `Sphere51a/Rental/RentalRegion.cs` | Custom region class |
| `Sphere51a/Gumps/RentalManagerGump.cs` | GM management gump |
| `Sphere51a/Gumps/RentalSignGump.cs` | Player rental gump |
| `Sphere51a/Gumps/RentalReadmeGump.cs` | GM help pages |
| `Sphere51a/Gumps/RentalHelpGump.cs` | Player help pages |
| `Sphere51a/Commands/RentalCommands.cs` | Admin commands |
| `Sphere51a/Targets/RentalTargets.cs` | Target handlers |

---

## 13. Change Log

### v1.0.0 - 2025-01-02 (Initial Specification)
- Complete rental building system design
- GM management gump with README
- Player sign gump
- Auto-payment and eviction system
- Co-tenant support
- Database schema
- Full implementation outline