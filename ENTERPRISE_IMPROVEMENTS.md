# Enterprise-Grade USB Device Manager - Improvement Roadmap

## Executive Summary

This document outlines comprehensive improvements to transform USBView into the ultimate enterprise-grade USB device management solution for Linux. The enhancements focus on complete device visibility, advanced configuration capabilities, power management, security controls, and enterprise integration.

---

## 1. Enhanced Device Information Discovery

### 1.1 Complete USB Descriptor Parsing
**Current State:** Basic descriptor information is displayed (vendor ID, product ID, speed, configurations).

**Improvements:**
- [ ] **Full Descriptor Tree Display**
  - Device Qualifier Descriptors
  - Other Speed Configuration Descriptors
  - Device Capability Descriptors (USB 2.0 Extension, SuperSpeed, Container ID, Platform)
  - BOS (Binary Object Store) descriptors
  - String descriptor tables with all language IDs
  
- [ ] **Extended Sysfs Attributes**
  ```c
  // Add to sysfs.c - device_parse()
  device->authorized       = sysfs_int(dir, "authorized", 10);
  device->autosuspend      = sysfs_int(dir, "autosuspend_delay_ms", 10);
  device->avoidReset       = sysfs_int(dir, "avoid_reset_quirk", 10);
  device->configuration    = sysfs_string(dir, "configuration");
  device->devnum           = sysfs_int(dir, "devnum", 10);
  device->interface        = sysfs_string(dir, "interface");
  device->ltmCapable       = sysfs_int(dir, "ltm_capable", 10);
  device->maxPower         = sysfs_int(dir, "bMaxPower", 10);
  device->powerWakeup      = sysfs_string(dir, "power/wakeup");
  device->powerControl     = sysfs_string(dir, "power/control");
  device->runtimeStatus    = sysfs_string(dir, "power/runtime_status");
  device->usbVersion       = sysfs_string(dir, "version");
  device->driverBinding    = sysfs_string(dir, "driver/bind");
  device->uevent           = parse_uevent(dir);  // Full uevent parsing
  ```

- [ ] **Hardware-Specific Information**
  - PHY rate and link speed negotiation
  - Lane count for USB4/Thunderbolt
  - Cable detection and quality metrics
  - Port controller information
  - PCIe tunneling details for USB4

### 1.2 Real-Time Monitoring Dashboard
**New Feature:** Live monitoring panel showing:
- [ ] Bandwidth utilization per bus/port (bytes/sec, packets/sec)
- [ ] Error counters (CRC errors, protocol errors, timeout counts)
- [ ] Power consumption readings (where supported by hardware)
- [ ] Temperature sensors (for hubs with thermal monitoring)
- [ ] Connection/disconnection event log with timestamps
- [ ] Transaction throughput graphs

### 1.3 Advanced Device Identification
**New Feature:** Enhanced device fingerprinting:
- [ ] USB-IF certified device database integration
- [ ] Known problematic devices database with workarounds
- [ ] Device class-specific detailed views:
  - Mass Storage: partition tables, SMART data, write protection status
  - HID: full report descriptor parsing, input/output feature reports
  - Audio: audio class descriptors, sample rates, channel maps
  - Video: UVC descriptors, supported formats, frame rates
  - CDC/Network: MAC address, network statistics, connection type
- [ ] Vendor-specific extension support via plugin architecture

---

## 2. Advanced Configuration Management

### 2.1 Configuration Selection & Switching
**New Feature:** Active configuration management:
```c
// New file: config_manager.c
int usb_set_configuration(int busNum, int devNum, int configValue);
int usb_get_active_configuration(int busNum, int devNum);
int usb_list_configurations(int busNum, int devNum, struct ConfigDesc **configs);
```
- [ ] View all available configurations for a device
- [ ] Switch between configurations dynamically
- [ ] Save preferred configuration per device (persisted across reconnects)
- [ ] Auto-apply saved configurations on device connect
- [ ] Configuration profiles for different use cases

### 2.2 Interface & Endpoint Control
**New Feature:** Fine-grained interface management:
- [ ] Enable/disable individual interfaces
- [ ] Alternate setting selection for interfaces
- [ ] Endpoint stall recovery tools
- [ ] Endpoint halt/clear operations
- [ ] Custom endpoint packet size configuration (where supported)

### 2.3 Driver Binding Management
**New Feature:** Advanced driver control:
```bash
# Expose via UI:
echo -n "0000:00:14.0" > /sys/bus/usb/drivers/usb/unbind
echo -n "0000:00:14.0" > /sys/bus/usb/drivers/xhci_hcd/bind
```
- [ ] Unbind/rebind drivers without reboot
- [ ] Override default driver with specific driver
- [ ] Bind to usbfs/libusb for userspace access
- [ ] Driver blacklisting/whitelisting per device
- [ ] Test driver loading with mock devices

### 2.4 Quirks & Workarounds Management
**New Feature:** USB quirk configuration interface:
- [ ] GUI for setting USB_QUIRK_* flags per device
- [ ] Common quirks with descriptions:
  - LPM (Link Power Management) quirks
  - Reset quirks
  - Suspend quirks
  - Buffer size quirks
  - High-speed inaccuracy quirks
- [ ] Persistent quirk storage in `/etc/usbview/quirks.conf`
- [ ] Import/export quirk configurations

---

## 3. Power Management Suite

### 3.1 Granular Power Control
**New Feature:** Per-device power management:
```c
// New file: power_manager.c
typedef enum {
    POWER_CONTROL_ON = 0,      // Always on
    POWER_CONTROL_AUTO,        // Autosuspend allowed
    POWER_CONTROL_SUSPEND,     // Force suspend
} power_control_t;

int usb_set_power_control(const char *devpath, power_control_t control);
int usb_get_power_control(const char *devpath);
int usb_set_autosuspend_delay(const char *devpath, int delay_ms);
int usb_get_autosuspend_delay(const char *devpath);
int usb_set_wakeup_enabled(const char *devpath, gboolean enabled);
gboolean usb_get_wakeup_enabled(const char *devpath);
```

- [ ] **Power Policy Profiles:**
  - Maximum Performance (all devices always on)
  - Balanced (selective autosuspend)
  - Power Savings (aggressive autosuspend)
  - Custom per-device policies

- [ ] **Runtime Power Management:**
  - Configure autosuspend delay per device (100ms - 30000ms)
  - Force immediate suspend/resume
  - View runtime PM status (active/suspended/unknown)
  - Prevent autosuspend for critical devices

- [ ] **Wake-up Configuration:**
  - Enable/disable wake-up capability per device
  - View wake-up event history
  - Configure wake-up triggers (connect, disconnect, activity)

### 3.2 Power Budget Analysis
**New Feature:** USB power domain visualization:
- [ ] Show power budget per USB host controller
- [ ] Track allocated vs available power per hub
- [ ] Warn when device requests exceed port limits
- [ ] Calculate total system USB power consumption
- [ ] Historical power usage graphs

### 3.3 Selective Suspend Control
**New Feature:** USB selective suspend management:
- [ ] Global enable/disable for USB selective suspend
- [ ] Per-port suspend control on supported hubs
- [ ] Scheduled suspend/resume times
- [ ] Activity-based suspend policies

---

## 4. Speed & Performance Optimization

### 4.1 Link Speed Management
**New Feature:** USB speed negotiation control:
```c
// New file: speed_manager.c
typedef enum {
    SPEED_AUTO = 0,
    SPEED_LOW = 1,
    SPEED_FULL = 12,
    SPEED_HIGH = 480,
    SPEED_SUPER = 5000,
    SPEED_SUPER_PLUS = 10000,
} usb_speed_t;

int usb_force_speed(const char *devpath, usb_speed_t speed);
usb_speed_t usb_get_current_speed(const char *devpath);
int usb_get_supported_speeds(const char *devpath);
```

- [ ] Force specific USB speed for compatibility testing
- [ ] View negotiated speed vs maximum capable speed
- [ ] Identify speed-limiting factors (cable, port, device)
- [ ] Speed transition logging and analysis

### 4.2 Link Power Management (LPM)
**New Feature:** USB 2.0/3.x LPM configuration:
- [ ] Enable/disable LPM per device
- [ ] Configure LPM timeout values
- [ ] View LPM exit latency statistics
- [ ] BESL (Best Effort Service Latency) tuning

### 4.3 Buffer & Transfer Optimization
**New Feature:** Transfer parameter tuning:
- [ ] Adjust URB (USB Request Block) buffer sizes
- [ ] Configure transfer queue depths
- [ ] Set interrupt interval overrides
- [ ] Isochronous bandwidth allocation control

### 4.4 Performance Benchmarking
**New Feature:** Built-in USB performance tests:
- [ ] Sequential read/write speed tests (for mass storage)
- [ ] Latency measurements (for HID/audio devices)
- [ ] Sustained transfer rate monitoring
- [ ] Compare against theoretical maximums
- [ ] Generate performance reports

---

## 5. Enterprise Management Features

### 5.1 Device Authorization & Security
**New Feature:** USB device access control:
```c
// New file: security_manager.c
typedef enum {
    AUTH_POLICY_ALLOW = 0,
    AUTH_POLICY_DENY,
    AUTH_POLICY_REQUIRE_APPROVAL,
} auth_policy_t;

int usb_set_device_authorization(const char *devpath, gboolean authorized);
int usb_set_default_policy(auth_policy_t policy);
int usb_add_device_to_whitelist(uint16_t vendor, uint16_t product);
int usb_add_device_to_blacklist(uint16_t vendor, uint16_t product);
```

- [ ] **Device Authorization System:**
  - Require approval for new devices (via D-Bus notification)
  - Auto-authorize known devices
  - Auto-reject blacklisted devices
  - Time-limited authorization grants

- [ ] **Policy-Based Access Control:**
  - Define policies by device class, vendor, serial number
  - Role-based access (admin, power user, standard user)
  - Integration with PAM for authentication
  - Audit log of all authorization decisions

- [ ] **Port-Level Security:**
  - Disable specific physical ports
  - Port-level authorization policies
  - Tamper detection (unexpected device on secured port)

### 5.2 Inventory & Asset Management
**New Feature:** Device inventory tracking:
- [ ] Maintain database of all ever-connected devices
- [ ] Track device lifecycle (first seen, last seen, total connect time)
- [ ] Assign custom labels/names to devices
- [ ] Group devices by location, department, user
- [ ] Export inventory to CSV/JSON/XML
- [ ] Integration with IT asset management systems

### 5.3 Compliance & Reporting
**New Feature:** Regulatory compliance tools:
- [ ] Generate compliance reports (security policies enforced)
- [ ] USB device usage reports by user/time period
- [ ] Data exfiltration risk assessment
- [ ] Scheduled report generation and email delivery
- [ ] Integration with SIEM systems

### 5.4 Remote Management
**New Feature:** Remote USB device administration:
- [ ] D-Bus API for remote control
- [ ] Web-based management interface (optional)
- [ ] SSH-compatible CLI tool (`usbviewctl`)
- [ ] Centralized management server for multiple endpoints
- [ ] Push policies to managed devices

---

## 6. User Interface Enhancements

### 6.1 Modern GTK4/Libadwaita Port
**Current State:** Uses GTK3 (or earlier)

**Improvements:**
- [ ] Migrate to GTK4 and Libadwaita for modern look
- [ ] Dark mode support
- [ ] Responsive design for various screen sizes
- [ ] Touch-friendly controls for tablets
- [ ] Animated transitions and visual feedback

### 6.2 Multi-Pane Advanced View
**New Layout:**
```
┌─────────────────────────────────────────────────────────────┐
│ Menu Bar: File  Edit  View  Devices  Tools  Help            │
├──────────┬──────────────────────────────────────────────────┤
│          │  Tab 1: Overview  Tab 2: Descriptors  Tab 3: ... │
│  Device  ├──────────────────────────────────────────────────┤
│   Tree   │  ┌────────────────────────────────────────────┐  │
│  (Left)  │  │  Detailed Property Grid                    │  │
│          │  │  ┌─────────────┬──────────────────────────┐ │  │
│  Filters │  │  │ Property    │ Value                    │ │  │
│          │  │  ├─────────────┼──────────────────────────┤ │  │
│  Search  │  │  │ Bus Number  │ 3                        │ │  │
│          │  │  │ Device Addr │ 14                       │ │  │
│          │  │  │ ...         │ ...                      │ │  │
│          │  │  └─────────────┴──────────────────────────┘ │  │
│          │  └────────────────────────────────────────────┘  │
│          │                                                  │
│          │  [Apply] [Reset] [Export] [Refresh]              │
└──────────┴──────────────────────────────────────────────────┘
```

### 6.3 Interactive Topology Map
**New Feature:** Visual USB topology diagram:
- [ ] Graphical representation of USB tree
- [ ] Color-coded by speed, power state, or device class
- [ ] Click-to-select devices
- [ ] Zoom and pan controls
- [ ] Export as PNG/SVG

### 6.4 Smart Search & Filtering
**New Feature:** Advanced device discovery:
- [ ] Real-time search across all device properties
- [ ] Filter by:
  - Vendor/Product ID
  - Device class/subclass
  - Speed
  - Power state
  - Driver name
  - Serial number pattern
  - Connection time
  - Authorization status
- [ ] Save filter presets
- [ ] Regex support for advanced searches

### 6.5 Context-Aware Actions
**New Feature:** Dynamic action menus:
- [ ] Right-click context menu with relevant actions per device type
- [ ] Mass Storage: safely eject, benchmark, view partitions
- [ ] HID: test inputs, view report descriptor
- [ ] Audio: test audio, configure sample rate
- [ ] Network: view stats, configure settings
- [ ] Composite devices: per-interface actions

---

## 7. Automation & Scripting

### 7.1 Command-Line Interface
**New Tool:** `usbviewctl` - Full-featured CLI:
```bash
# Device enumeration
usbviewctl list [--tree|--flat|--json]
usbviewctl show <bus>:<dev> [--full|--descriptors|--properties]

# Configuration
usbviewctl config <bus>:<dev> set <config_num>
usbviewctl interface <bus>:<dev>:<if> enable|disable

# Power management
usbviewctl power <bus>:<dev> control on|auto|suspend
usbviewctl power <bus>:<dev> wakeup enable|disable
usbviewctl power <bus>:<dev> autosuspend-delay <ms>

# Speed control
usbviewctl speed <bus>:<dev> force <speed>
usbviewctl speed <bus>:<dev> info

# Security
usbviewctl authorize <bus>:<dev> allow|deny
usbviewctl policy add --vendor 0x1234 --action allow
usbviewctl policy list

# Monitoring
usbviewctl monitor [--events|--bandwidth|--errors]
usbviewctl watch --connect --disconnect --exec "/path/to/script"

# Diagnostics
usbviewctl diagnose <bus>:<dev>
usbviewctl test <bus>:<dev> --benchmark --latency
```

### 7.2 Event-Driven Automation
**New Feature:** Device event hooks:
```c
// New file: event_system.c
typedef enum {
    EVENT_DEVICE_ADDED = 0,
    EVENT_DEVICE_REMOVED,
    EVENT_DEVICE_AUTHORIZED,
    EVENT_DEVICE_UNAUTHORIZED,
    EVENT_POWER_STATE_CHANGED,
    EVENT_SPEED_CHANGED,
    EVENT_ERROR_DETECTED,
} usb_event_type_t;

typedef void (*event_handler_t)(usb_event_type_t type, 
                                 const char *devpath,
                                 gpointer user_data);

int usb_register_event_handler(usb_event_type_t events,
                                event_handler_t handler,
                                gpointer user_data);
```

- [ ] Execute scripts on device connect/disconnect
- [ ] Trigger actions based on device properties
- [ ] Send notifications (desktop, email, webhook)
- [ ] Log events to syslog/journald
- [ ] Integration with systemd service activation

### 7.3 Configuration as Code
**New Feature:** Declarative USB configuration:
```yaml
# /etc/usbview/devices.yaml
devices:
  - vendor: 0x0781
    product: 0x5581
    name: "SanDisk Ultra USB"
    policy:
      authorized: true
      power_control: auto
      autosuspend_delay: 2000
      wakeup: false
      
  - vendor: 0x046d
    product: 0xc52b
    name: "Logitech Wireless Receiver"
    policy:
      authorized: true
      power_control: on  # Never suspend wireless devices
      wakeup: true
      
ports:
  - path: "usb1/1-1"
    policy:
      max_speed: high
      authorized_classes: [hid, mass_storage]
```

---

## 8. Logging, Auditing & Diagnostics

### 8.1 Comprehensive Event Logging
**New Feature:** Detailed audit trail:
```c
// New file: audit_log.c
typedef struct {
    timestamp_t timestamp;
    gchar *event_type;
    gchar *device_path;
    gchar *user_id;
    gchar *action;
    gchar *result;
    gchar *details;
} audit_entry_t;

int audit_log_event(const gchar *event_type,
                    const gchar *device_path,
                    const gchar *action,
                    gboolean success,
                    const gchar *details);
```

- [ ] Log all device connections/disconnections
- [ ] Record all configuration changes
- [ ] Track authorization decisions
- [ ] Capture error conditions
- [ ] Tamper-evident log storage
- [ ] Log rotation and archival

### 8.2 Diagnostic Tools Suite
**New Feature:** Built-in troubleshooting tools:
- [ ] **USB Protocol Analyzer Lite:**
  - Capture USB traffic (with appropriate permissions)
  - Decode common USB protocols
  - Identify protocol violations
  
- [ ] **Device Health Check:**
  - Run diagnostic tests on selected device
  - Check for common issues (high error rates, power problems)
  - Suggest fixes based on symptoms
  
- [ ] **System USB Health:**
  - Check host controller status
  - Verify kernel driver versions
  - Detect known hardware errata
  - Review kernel log for USB errors

### 8.3 Debug Mode & Verbose Output
**New Feature:** Enhanced debugging capabilities:
- [ ] Enable verbose logging to file
- [ ] Export complete device state snapshot
- [ ] Generate bug report package
- [ ] Interactive device property editor (debug only)

---

## 9. Plugin Architecture

### 9.1 Extensible Plugin System
**New Feature:** Third-party extensibility:
```c
// New file: plugin_api.h
typedef struct {
    const char *name;
    const char *version;
    const char *description;
    
    int (*init)(void);
    void (*cleanup)(void);
    
    // Device-specific extensions
    gboolean (*supports_device)(struct Device *dev);
    GtkWidget *(*create_device_view)(struct Device *dev);
    int (*device_action)(struct Device *dev, const char *action);
    
    // Menu extensions
    GList *(*get_menu_items)(void);
} USBViewPlugin;
```

- [ ] Device class-specific plugins (storage, audio, video, etc.)
- [ ] Vendor-specific enhancement plugins
- [ ] Custom action plugins
- [ ] Export format plugins
- [ ] Notification backend plugins

### 9.2 Python Scripting Support
**New Feature:** Embedded Python interpreter:
- [ ] Write custom device handlers in Python
- [ ] Automate complex workflows
- [ ] Integrate with external APIs
- [ ] Rapid prototyping of new features

---

## 10. Integration & Interoperability

### 10.1 System Integration
**New Features:**
- [ ] **udev Rule Generator:**
  - Create udev rules from GUI
  - Test rules before applying
  - Manage existing rules
  
- [ ] **systemd Integration:**
  - Activate services on device connect
  - Device-specific service instances
  - Resource management via systemd slices
  
- [ ] **Desktop Environment Integration:**
  - Freedesktop notifications
  - System tray indicator
  - GNOME/KDE settings panels
  - Unity launcher integration

### 10.2 External Tool Integration
**New Features:**
- [ ] Launch external tools for specific device types:
  - `lsusb -v` detailed output
  - `usbmon` traffic capture
  - GParted for storage devices
  - PulseAudio/pavucontrol for audio
  - Wireshark for network adapters
  
- [ ] **Wireshark Integration:**
  - One-click start capture for device
  - Import/export pcap files
  - Correlate USB events with traffic

### 10.3 Virtualization Support
**New Feature:** USB passthrough management:
- [ ] View devices available for VM passthrough
- [ ] Configure libvirt USB redirection
- [ ] Manage QEMU USB device assignments
- [ ] Release device from host for VM use

---

## 11. Technical Implementation Priorities

### Phase 1: Foundation (Months 1-2)
1. Migrate to GTK4/Libadwaita
2. Implement enhanced sysfs parsing (Section 1.1)
3. Add basic power management controls (Section 3.1)
4. Create CLI tool skeleton (Section 7.1)

### Phase 2: Core Features (Months 3-4)
1. Device authorization system (Section 5.1)
2. Configuration management (Section 2.1, 2.2)
3. Advanced UI with multi-pane view (Section 6.2)
4. Event system and automation (Section 7.2)

### Phase 3: Enterprise Features (Months 5-6)
1. Inventory and asset management (Section 5.2)
2. Audit logging (Section 8.1)
3. Remote management APIs (Section 5.4)
4. Policy engine (Section 5.1)

### Phase 4: Advanced Features (Months 7-8)
1. Plugin architecture (Section 9)
2. Performance benchmarking (Section 4.4)
3. Visual topology map (Section 6.3)
4. Python scripting (Section 9.2)

### Phase 5: Polish & Integration (Months 9-10)
1. System integrations (Section 10)
2. Comprehensive testing
3. Documentation
4. Packaging for major distributions

---

## 12. Security Considerations

### 12.1 Privilege Separation
- [ ] Run core daemon as root (minimal privileges)
- [ ] UI runs as regular user
- [ ] D-Bus for secure communication
- [ ] Polkit for privilege escalation prompts

### 12.2 Secure Defaults
- [ ] Deny-by-default for new devices (enterprise mode)
- [ ] All changes logged and auditable
- [ ] No persistent changes without explicit confirmation
- [ ] Regular security updates for dependencies

### 12.3 Access Control
- [ ] Unix group-based access control
- [ ] Fine-grained capabilities (view-only, configure, authorize)
- [ ] Session-based permissions
- [ ] Integration with enterprise identity providers

---

## 13. Performance Requirements

### 13.1 Responsiveness
- UI refresh < 100ms for typical operations
- Device enumeration < 2 seconds for 50+ devices
- Background monitoring overhead < 1% CPU

### 13.2 Scalability
- Support 100+ simultaneously connected devices
- Handle 1000+ devices in inventory database
- Multi-instance support for large deployments

### 13.3 Resource Usage
- Memory footprint < 50MB for UI
- Daemon memory < 20MB
- Minimal disk I/O during normal operation

---

## 14. Documentation Requirements

### 14.1 User Documentation
- Comprehensive user manual (PDF, HTML, epub)
- Quick-start guide
- Video tutorials
- In-application help system

### 14.2 Administrator Guide
- Deployment guide
- Security hardening guide
- Policy configuration reference
- Troubleshooting guide

### 14.3 Developer Documentation
- API reference
- Plugin development guide
- Contribution guidelines
- Architecture documentation

---

## 15. Testing Strategy

### 15.1 Automated Testing
- Unit tests for all core functions (>80% coverage)
- Integration tests with real USB devices
- UI automated testing with dogtail/squish
- Continuous integration pipeline

### 15.2 Hardware Compatibility Lab
- Test matrix covering major USB device classes
- Multiple host controllers (Intel, AMD, ASMedia, etc.)
- Various kernel versions (LTS + latest)
- Different desktop environments

### 15.3 Beta Program
- Community beta testing program
- Enterprise partner early access
- Feedback collection and triage process

---

## Conclusion

This roadmap transforms USBView from a simple USB device viewer into a comprehensive enterprise-grade USB device management platform. The improvements span device discovery, configuration, power management, security, automation, and integration—making it the definitive USB management tool for Linux.

Key differentiators:
- **Complete Visibility:** Every USB descriptor, attribute, and status exposed
- **Granular Control:** Per-device, per-interface, per-endpoint configuration
- **Enterprise Ready:** Security policies, audit logs, remote management
- **Extensible:** Plugin architecture and scripting support
- **Modern UX:** GTK4 interface with intelligent workflows

Implementation should follow the phased approach, delivering value incrementally while building toward the complete vision.

---

## Appendix A: New File Structure

```
/workspace/
├── src/
│   ├── main.c
│   ├── interface.c              # Updated for GTK4
│   ├── callbacks.c              # Extended callbacks
│   ├── usbtree.c
│   ├── usbtree.h
│   ├── sysfs.c                  # Enhanced parsing
│   ├── sysfs.h
│   ├── config_manager.c         # NEW: Configuration management
│   ├── config_manager.h
│   ├── power_manager.c          # NEW: Power management
│   ├── power_manager.h
│   ├── speed_manager.c          # NEW: Speed control
│   ├── speed_manager.h
│   ├── security_manager.c       # NEW: Authorization/security
│   ├── security_manager.h
│   ├── event_system.c           # NEW: Event handling
│   ├── event_system.h
│   ├── audit_log.c              # NEW: Audit logging
│   ├── audit_log.h
│   ├── plugin_api.c             # NEW: Plugin system
│   ├── plugin_api.h
│   ├── cli/                     # NEW: Command-line tool
│   │   ├── usbviewctl.c
│   │   └── commands.c
│   └── plugins/                 # NEW: Built-in plugins
│       ├── storage_plugin.c
│       ├── audio_plugin.c
│       └── hid_plugin.c
├── ui/
│   ├── window_main.ui           # NEW: GTK4 UI definitions
│   ├── dialog_config.ui
│   ├── dialog_power.ui
│   ├── dialog_security.ui
│   └── icons/
├── data/
│   ├── usb-ids                  # USB ID database
│   └── quirks.conf.example
├── docs/
│   ├── user-guide.md
│   ├── admin-guide.md
│   └── developer-guide.md
└── tests/
    ├── unit/
    ├── integration/
    └── fixtures/
```

---

## Appendix B: Key Dependencies

### Required
- GTK4 >= 4.0
- Libadwaita >= 1.0
- GLib >= 2.68
- systemd libudev
- D-Bus (GDBus)
- Polkit (for privilege escalation)

### Optional
- Python3 (for scripting)
- JavaScript/GJS (for alternative scripting)
- libpcap (for USB traffic capture)
- SQLite3 (for inventory database)
- libvirt (for VM integration)
- WebKitGTK (for web-based management)

---

*Document Version: 1.0*
*Last Updated: 2024*
*Author: Enterprise USB Management Initiative*
