# Quick-Start Implementation Guide

This guide provides actionable code examples to begin implementing the most impactful enterprise features for USBView.

---

## Phase 1: Enhanced Device Information (Start Here)

### Step 1: Extend sysfs.c with Power Management Attributes

Add these fields to the `struct Device` in **sysfs.h**:

```c
// Add to struct Device in sysfs.h, after serialNumber:
    gchar           *powerControl;        // "on", "auto", "suspend"
    gchar           *powerWakeup;         // "enabled", "disabled"
    gchar           *runtimeStatus;       // "active", "suspended", "unknown"
    gint            autosuspendDelay;     // milliseconds
    gboolean        authorized;           // device authorization status
    gchar           *interface;           // interface description
    gboolean        ltmCapable;           // Link Training Mechanism capable
```

### Step 2: Parse Additional Sysfs Attributes in sysfs.c

Add this function to **sysfs.c** after the existing helper functions:

```c
// Add to sysfs.c around line 150, after sysfs_int()

static gboolean sysfs_bool(const char *dir, const char *filename)
{
    char *string = sysfs_string(dir, filename);
    
    if (!string)
        return FALSE;
    
    gboolean value = (strcmp(string, "enabled") == 0 || 
                      strcmp(string, "1") == 0);
    
    g_free(string);
    return value;
}
```

Then update the `device_parse()` function to read these new attributes:

```c
// Find device_parse() in sysfs.c (around line 669)
// Add these lines after reading serialNumber (around line 730):

    device->authorized        = sysfs_int(dir, "authorized", 10);
    device->autosuspendDelay  = sysfs_int(dir, "power/autosuspend_delay_ms", 10);
    device->powerControl      = sysfs_string(dir, "power/control");
    device->powerWakeup       = sysfs_string(dir, "power/wakeup");
    device->runtimeStatus     = sysfs_string(dir, "power/runtime_status");
    device->ltmCapable        = sysfs_int(dir, "ltm_capable", 10);
    device->interface         = sysfs_string(dir, "interface");
    
    // Parse uevent for additional metadata
    char uevent_path[PATH_MAX];
    snprintf(uevent_path, PATH_MAX, "%s/uevent", dir);
    // Could parse uevent file here for MODALIAS and other info
```

### Step 3: Display Power Information in usbtree.c

Update `PopulateListBox()` in **usbtree.c** to show power management info:

```c
// In usbtree.c, PopulateListBox() function
// After displaying bandwidth info (around line 128), add:

    /* add power management information */
    if (device->powerControl != NULL) {
        sprintf (string, "\nPower Control: %s", device->powerControl);
        gtk_text_buffer_insert_at_cursor(textDescriptionBuffer, string, strlen(string));
    }
    
    if (device->powerWakeup != NULL) {
        sprintf (string, "\nWake-up: %s", device->powerWakeup);
        gtk_text_buffer_insert_at_cursor(textDescriptionBuffer, string, strlen(string));
    }
    
    if (device->runtimeStatus != NULL) {
        sprintf (string, "\nRuntime Status: %s", device->runtimeStatus);
        gtk_text_buffer_insert_at_cursor(textDescriptionBuffer, string, strlen(string));
    }
    
    if (device->autosuspendDelay > 0) {
        sprintf (string, "\nAutosuspend Delay: %d ms", device->autosuspendDelay);
        gtk_text_buffer_insert_at_cursor(textDescriptionBuffer, string, strlen(string));
    }
    
    sprintf (string, "\nAuthorized: %s", device->authorized ? "Yes" : "No");
    gtk_text_buffer_insert_at_cursor(textDescriptionBuffer, string, strlen(string));
    
    if (device->ltmCapable) {
        gtk_text_buffer_insert_at_cursor(textDescriptionBuffer, "\nLPM Capable: Yes", 16);
    }
```

---

## Phase 2: Power Management Controls

### Step 4: Create power_manager.c

Create a new file **power_manager.c**:

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * power_manager.c - USB Power Management Controls
 * Copyright (c) 2024 by Enterprise USB Management Initiative
 */

#ifdef HAVE_CONFIG_H
    #include <config.h>
#endif

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <glib.h>
#include <sys/stat.h>

#include "power_manager.h"

#define SYSFS_USB_PATH "/sys/bus/usb/devices"

/**
 * Write a string to a sysfs attribute
 * Returns: 0 on success, -1 on error
 */
static int write_sysfs_attr(const char *devpath, const char *attr, const char *value)
{
    char filepath[512];
    int fd, ret;
    
    snprintf(filepath, sizeof(filepath), "%s/%s", devpath, attr);
    
    fd = open(filepath, O_WRONLY);
    if (fd < 0) {
        g_warning("Failed to open %s: %s", filepath, strerror(errno));
        return -1;
    }
    
    ret = write(fd, value, strlen(value));
    close(fd);
    
    if (ret < 0) {
        g_warning("Failed to write to %s: %s", filepath, strerror(errno));
        return -1;
    }
    
    return 0;
}

/**
 * Read a string from a sysfs attribute
 * Returns: newly allocated string (must be freed), or NULL on error
 */
static char* read_sysfs_attr(const char *devpath, const char *attr)
{
    char filepath[512];
    char buffer[256];
    int fd, count;
    
    snprintf(filepath, sizeof(filepath), "%s/%s", devpath, attr);
    
    fd = open(filepath, O_RDONLY);
    if (fd < 0) {
        g_warning("Failed to open %s: %s", filepath, strerror(errno));
        return NULL;
    }
    
    count = read(fd, buffer, sizeof(buffer) - 1);
    close(fd);
    
    if (count < 0) {
        g_warning("Failed to read from %s: %s", filepath, strerror(errno));
        return NULL;
    }
    
    buffer[count] = '\0';
    // Strip trailing newline
    if (count > 0 && buffer[count-1] == '\n')
        buffer[count-1] = '\0';
    
    return g_strdup(buffer);
}

/**
 * Set power control mode for a USB device
 * @param devpath: Device path (e.g., "/sys/bus/usb/devices/1-1")
 * @param control: Power control mode (POWER_CONTROL_ON/AUTO/SUSPEND)
 * @return: 0 on success, -1 on error
 */
int usb_set_power_control(const char *devpath, power_control_t control)
{
    const char *value;
    
    switch (control) {
        case POWER_CONTROL_ON:
            value = "on";
            break;
        case POWER_CONTROL_AUTO:
            value = "auto";
            break;
        case POWER_CONTROL_SUSPEND:
            value = "suspend";
            break;
        default:
            g_warning("Invalid power control mode: %d", control);
            return -1;
    }
    
    return write_sysfs_attr(devpath, "power/control", value);
}

/**
 * Get current power control mode
 * @param devpath: Device path
 * @return: Current power control mode, or -1 on error
 */
power_control_t usb_get_power_control(const char *devpath)
{
    char *value;
    power_control_t result = -1;
    
    value = read_sysfs_attr(devpath, "power/control");
    if (!value)
        return -1;
    
    if (strcmp(value, "on") == 0)
        result = POWER_CONTROL_ON;
    else if (strcmp(value, "auto") == 0)
        result = POWER_CONTROL_AUTO;
    else if (strcmp(value, "suspend") == 0)
        result = POWER_CONTROL_SUSPEND;
    
    g_free(value);
    return result;
}

/**
 * Set autosuspend delay in milliseconds
 * @param devpath: Device path
 * @param delay_ms: Delay in milliseconds (-1 to disable, 0 for immediate)
 * @return: 0 on success, -1 on error
 */
int usb_set_autosuspend_delay(const char *devpath, int delay_ms)
{
    char value[32];
    
    snprintf(value, sizeof(value), "%d", delay_ms);
    return write_sysfs_attr(devpath, "power/autosuspend_delay_ms", value);
}

/**
 * Get autosuspend delay
 * @param devpath: Device path
 * @return: Delay in milliseconds, or -1 on error
 */
int usb_get_autosuspend_delay(const char *devpath)
{
    char *value;
    int result;
    
    value = read_sysfs_attr(devpath, "power/autosuspend_delay_ms");
    if (!value)
        return -1;
    
    result = atoi(value);
    g_free(value);
    return result;
}

/**
 * Enable or disable wake-up capability
 * @param devpath: Device path
 * @param enabled: TRUE to enable wake-up, FALSE to disable
 * @return: 0 on success, -1 on error
 */
int usb_set_wakeup_enabled(const char *devpath, gboolean enabled)
{
    return write_sysfs_attr(devpath, "power/wakeup", enabled ? "enabled" : "disabled");
}

/**
 * Check if wake-up is enabled
 * @param devpath: Device path
 * @return: TRUE if enabled, FALSE otherwise
 */
gboolean usb_get_wakeup_enabled(const char *devpath)
{
    char *value;
    gboolean result;
    
    value = read_sysfs_attr(devpath, "power/wakeup");
    if (!value)
        return FALSE;
    
    result = (strcmp(value, "enabled") == 0);
    g_free(value);
    return result;
}

/**
 * Get runtime power management status
 * @param devpath: Device path
 * @return: Runtime status string (must be freed), or NULL on error
 */
char* usb_get_runtime_status(const char *devpath)
{
    return read_sysfs_attr(devpath, "power/runtime_status");
}

/**
 * Force device suspend (if supported)
 * @param devpath: Device path
 * @return: 0 on success, -1 on error
 */
int usb_force_suspend(const char *devpath)
{
    // First set to auto mode
    if (usb_set_power_control(devpath, POWER_CONTROL_AUTO) < 0)
        return -1;
    
    // Writing "suspend" forces immediate suspend
    return write_sysfs_attr(devpath, "power/control", "suspend");
}

/**
 * Force device resume (if suspended)
 * @param devpath: Device path
 * @return: 0 on success, -1 on error
 */
int usb_force_resume(const char *devpath)
{
    // Setting to "on" will force resume if suspended
    return usb_set_power_control(devpath, POWER_CONTROL_ON);
}

/**
 * Get device authorization status
 * @param devpath: Device path
 * @return: TRUE if authorized, FALSE otherwise
 */
gboolean usb_is_authorized(const char *devpath)
{
    char *value;
    gboolean result;
    
    value = read_sysfs_attr(devpath, "authorized");
    if (!value)
        return FALSE;
    
    result = (strcmp(value, "1") == 0);
    g_free(value);
    return result;
}

/**
 * Set device authorization
 * @param devpath: Device path
 * @param authorized: TRUE to authorize, FALSE to deauthorize
 * @return: 0 on success, -1 on error
 */
int usb_set_authorization(const char *devpath, gboolean authorized)
{
    return write_sysfs_attr(devpath, "authorized", authorized ? "1" : "0");
}
```

### Step 5: Create power_manager.h

Create **power_manager.h**:

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * power_manager.h - USB Power Management Controls
 * Copyright (c) 2024 by Enterprise USB Management Initiative
 */

#ifndef __POWER_MANAGER_H
#define __POWER_MANAGER_H

#include <glib.h>

typedef enum {
    POWER_CONTROL_ON = 0,      // Always on, no autosuspend
    POWER_CONTROL_AUTO,        // Autosuspend allowed
    POWER_CONTROL_SUSPEND,     // Force suspend
} power_control_t;

// Power control functions
int usb_set_power_control(const char *devpath, power_control_t control);
power_control_t usb_get_power_control(const char *devpath);

// Autosuspend configuration
int usb_set_autosuspend_delay(const char *devpath, int delay_ms);
int usb_get_autosuspend_delay(const char *devpath);

// Wake-up control
int usb_set_wakeup_enabled(const char *devpath, gboolean enabled);
gboolean usb_get_wakeup_enabled(const char *devpath);

// Runtime status
char* usb_get_runtime_status(const char *devpath);

// Force power state changes
int usb_force_suspend(const char *devpath);
int usb_force_resume(const char *devpath);

// Device authorization
gboolean usb_is_authorized(const char *devpath);
int usb_set_authorization(const char *devpath, gboolean authorized);

#endif /* __POWER_MANAGER_H */
```

---

## Phase 3: Add Power Controls to UI

### Step 6: Update Makefile.am

Add the new source files to **Makefile.am**:

```makefile
usbview_SOURCES =               \
        main.c                  \
        interface.c             \
        callbacks.c             \
        usbtree.c usbtree.h     \
        sysfs.c sysfs.h         \
        power_manager.c power_manager.h \
        ccan/check_type/check_type.h    \
        ccan/str/str.h                  \
        ccan/str/str_debug.h            \
        ccan/config.h                   \
        ccan/container_of/container_of.h\
        ccan/list/list.h                \
        usbview_logo.xpm
```

### Step 7: Add Power Control Dialog

Create a simple power control dialog. Add this function to **callbacks.c**:

```c
// Add to callbacks.c

#include "power_manager.h"
#include <sys/stat.h>

// Helper function to get device path from bus/dev numbers
static char* get_device_sysfs_path(int busNum, int devNum)
{
    char path[512];
    DIR *dir;
    struct dirent *de;
    
    snprintf(path, sizeof(path), "/sys/bus/usb/devices");
    dir = opendir(path);
    if (!dir)
        return NULL;
    
    while ((de = readdir(dir))) {
        if (de->d_name[0] == '.')
            continue;
        
        char devpath[512];
        snprintf(devpath, sizeof(devpath), "%s/%s", path, de->d_name);
        
        // Read busnum and devnum
        FILE *f;
        int b, d;
        
        snprintf(devpath, sizeof(devpath), "%s/%s/busnum", path, de->d_name);
        f = fopen(devpath, "r");
        if (!f) continue;
        fscanf(f, "%d", &b);
        fclose(f);
        
        snprintf(devpath, sizeof(devpath), "%s/%s/devnum", path, de->d_name);
        f = fopen(devpath, "r");
        if (!f) continue;
        fscanf(f, "%d", &d);
        fclose(f);
        
        if (b == busNum && d == devNum) {
            closedir(dir);
            return g_strdup_printf("%s/%s", path, de->d_name);
        }
    }
    
    closedir(dir);
    return NULL;
}

void on_menuPowerControl_activate(GtkMenuItem *menuitem, gpointer user_data)
{
    // This would be called from a menu item
    // For now, just demonstrate the power manager API
    
    GtkTreeIter iter;
    GtkTreeModel *model;
    gint deviceAddr;
    
    GtkTreeSelection *select = gtk_tree_view_get_selection(GTK_TREE_VIEW(treeUSB));
    
    if (!gtk_tree_selection_get_selected(select, &model, &iter)) {
        gtk_show_about_dialog(GTK_WINDOW(windowMain),
                             "program-name", "USB Power Control",
                             "comments", "Please select a device first",
                             NULL);
        return;
    }
    
    gtk_tree_model_get(model, &iter, DEVICE_ADDR_COLUMN, &deviceAddr, -1);
    
    int busNum = deviceAddr & 0x00ff;
    int devNum = (deviceAddr >> 8) & 0xff;
    
    char *devpath = get_device_sysfs_path(busNum, devNum);
    if (!devpath) {
        gtk_show_about_dialog(GTK_WINDOW(windowMain),
                             "program-name", "USB Power Control",
                             "comments", "Could not find device sysfs path",
                             NULL);
        return;
    }
    
    // Example: Toggle power control between "on" and "auto"
    power_control_t current = usb_get_power_control(devpath);
    power_control_t new_control = (current == POWER_CONTROL_ON) ? 
                                   POWER_CONTROL_AUTO : POWER_CONTROL_ON;
    
    if (usb_set_power_control(devpath, new_control) == 0) {
        char msg[256];
        snprintf(msg, sizeof(msg), 
                 "Power control changed to: %s",
                 (new_control == POWER_CONTROL_ON) ? "Always On" : "Auto-suspend");
        
        gtk_show_about_dialog(GTK_WINDOW(windowMain),
                             "program-name", "USB Power Control",
                             "comments", msg,
                             NULL);
        
        // Refresh the display to show updated info
        LoadUSBTree(1);
    } else {
        gtk_show_about_dialog(GTK_WINDOW(windowMain),
                             "program-name", "USB Power Control Error",
                             "comments", "Failed to change power control (requires root privileges)",
                             NULL);
    }
    
    g_free(devpath);
}
```

---

## Phase 4: Create Command-Line Tool Skeleton

### Step 8: Create usbviewctl.c

Create **cli/usbviewctl.c**:

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * usbviewctl.c - Command-line USB device management tool
 * Copyright (c) 2024 by Enterprise USB Management Initiative
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <getopt.h>
#include <dirent.h>
#include <glib.h>

#include "../power_manager.h"

static void print_usage(void)
{
    printf("Usage: usbviewctl [command] [options]\n\n");
    printf("Commands:\n");
    printf("  list                    List all USB devices\n");
    printf("  show <bus>:<dev>        Show detailed device information\n");
    printf("  power <bus>:<dev> ...   Power management commands\n");
    printf("  authorize <bus>:<dev>   Authorize/deauthorize device\n");
    printf("\n");
    printf("Power subcommands:\n");
    printf("  control on|auto|suspend\n");
    printf("  wakeup enable|disable\n");
    printf("  delay <milliseconds>\n");
    printf("  status                  Show power status\n");
    printf("\n");
    printf("Examples:\n");
    printf("  usbviewctl list\n");
    printf("  usbviewctl show 1:5\n");
    printf("  usbviewctl power 1:5 control auto\n");
    printf("  usbviewctl power 1:5 wakeup disable\n");
    printf("  usbviewctl authorize 1:5 deny\n");
}

static char* find_device_path(int bus, int dev)
{
    static char path[512];
    DIR *dir = opendir("/sys/bus/usb/devices");
    if (!dir) return NULL;
    
    struct dirent *de;
    while ((de = readdir(dir))) {
        if (de->d_name[0] == '.') continue;
        
        char tmp[512];
        FILE *f;
        int b, d;
        
        snprintf(tmp, sizeof(tmp), "/sys/bus/usb/devices/%s/busnum", de->d_name);
        f = fopen(tmp, "r");
        if (!f) continue;
        fscanf(f, "%d", &b);
        fclose(f);
        
        snprintf(tmp, sizeof(tmp), "/sys/bus/usb/devices/%s/devnum", de->d_name);
        f = fopen(tmp, "r");
        if (!f) continue;
        fscanf(f, "%d", &d);
        fclose(f);
        
        if (b == bus && d == dev) {
            snprintf(path, sizeof(path), "/sys/bus/usb/devices/%s", de->d_name);
            closedir(dir);
            return path;
        }
    }
    
    closedir(dir);
    return NULL;
}

static void cmd_list(void)
{
    DIR *dir = opendir("/sys/bus/usb/devices");
    if (!dir) {
        fprintf(stderr, "Cannot access /sys/bus/usb/devices\n");
        exit(1);
    }
    
    printf("%-8s %-8s %-12s %-20s %s\n", 
           "Bus", "Dev", "Speed", "Manufacturer", "Product");
    printf("--------------------------------------------------------------\n");
    
    struct dirent *de;
    while ((de = readdir(dir))) {
        if (de->d_name[0] == '.') continue;
        
        char path[512], buf[256];
        FILE *f;
        int bus, dev, speed;
        
        snprintf(path, sizeof(path), "/sys/bus/usb/devices/%s/busnum", de->d_name);
        f = fopen(path, "r");
        if (!f) continue;
        fscanf(f, "%d", &bus);
        fclose(f);
        
        snprintf(path, sizeof(path), "/sys/bus/usb/devices/%s/devnum", de->d_name);
        f = fopen(path, "r");
        if (!f) continue;
        fscanf(f, "%d", &dev);
        fclose(f);
        
        snprintf(path, sizeof(path), "/sys/bus/usb/devices/%s/speed", de->d_name);
        f = fopen(path, "r");
        if (!f) continue;
        fscanf(f, "%d", &speed);
        fclose(f);
        
        const char *speed_str;
        switch(speed) {
            case 1: speed_str = "1.5M (Low)"; break;
            case 12: speed_str = "12M (Full)"; break;
            case 480: speed_str = "480M (High)"; break;
            case 5000: speed_str = "5G (Super)"; break;
            case 10000: speed_str = "10G (Super+)"; break;
            default: speed_str = "Unknown"; break;
        }
        
        snprintf(path, sizeof(path), "/sys/bus/usb/devices/%s/manufacturer", de->d_name);
        f = fopen(path, "r");
        if (f) {
            fgets(buf, sizeof(buf), f);
            buf[strcspn(buf, "\n")] = 0;
            fclose(f);
        } else {
            buf[0] = '\0';
        }
        char manufacturer[128];
        strncpy(manufacturer, buf, sizeof(manufacturer)-1);
        manufacturer[sizeof(manufacturer)-1] = '\0';
        
        snprintf(path, sizeof(path), "/sys/bus/usb/devices/%s/product", de->d_name);
        f = fopen(path, "r");
        if (f) {
            fgets(buf, sizeof(buf), f);
            buf[strcspn(buf, "\n")] = 0;
            fclose(f);
        } else {
            buf[0] = '\0';
        }
        char product[128];
        strncpy(product, buf, sizeof(product)-1);
        product[sizeof(product)-1] = '\0';
        
        printf("%-8d %-8d %-12s %-20.20s %s\n", 
               bus, dev, speed_str, manufacturer, product);
    }
    
    closedir(dir);
}

static void cmd_show(int bus, int dev)
{
    char *path = find_device_path(bus, dev);
    if (!path) {
        fprintf(stderr, "Device %d:%d not found\n", bus, dev);
        exit(1);
    }
    
    char file[512], buf[256];
    FILE *f;
    
    printf("Device: %d:%d\n", bus, dev);
    printf("Path: %s\n\n", path);
    
    const char *attrs[] = {
        "busnum", "devnum", "speed", "version",
        "bDeviceClass", "bDeviceSubClass", "bDeviceProtocol",
        "idVendor", "idProduct", "bcdDevice",
        "manufacturer", "product", "serial",
        "bMaxPower", "bNumConfigurations",
        "authorized", "power/control", "power/wakeup",
        "power/runtime_status", "power/autosuspend_delay_ms",
        NULL
    };
    
    for (int i = 0; attrs[i]; i++) {
        snprintf(file, sizeof(file), "%s/%s", path, attrs[i]);
        f = fopen(file, "r");
        if (f) {
            if (fgets(buf, sizeof(buf), f)) {
                buf[strcspn(buf, "\n")] = 0;
                printf("%-30s %s\n", attrs[i], buf);
            }
            fclose(f);
        }
    }
}

static void cmd_power(int bus, int dev, int argc, char **argv)
{
    if (argc < 1) {
        fprintf(stderr, "Power subcommand required\n");
        exit(1);
    }
    
    char *path = find_device_path(bus, dev);
    if (!path) {
        fprintf(stderr, "Device %d:%d not found\n", bus, dev);
        exit(1);
    }
    
    if (strcmp(argv[0], "status") == 0) {
        printf("Power Status for device %d:%d:\n", bus, dev);
        
        power_control_t ctrl = usb_get_power_control(path);
        printf("  Control: ");
        switch(ctrl) {
            case POWER_CONTROL_ON: printf("Always On\n"); break;
            case POWER_CONTROL_AUTO: printf("Auto-suspend\n"); break;
            case POWER_CONTROL_SUSPEND: printf("Suspended\n"); break;
            default: printf("Unknown\n"); break;
        }
        
        printf("  Wake-up: %s\n", usb_get_wakeup_enabled(path) ? "Enabled" : "Disabled");
        printf("  Autosuspend Delay: %d ms\n", usb_get_autosuspend_delay(path));
        
        char *runtime = usb_get_runtime_status(path);
        printf("  Runtime Status: %s\n", runtime ? runtime : "Unknown");
        g_free(runtime);
        
    } else if (strcmp(argv[0], "control") == 0 && argc >= 2) {
        power_control_t ctrl;
        if (strcmp(argv[1], "on") == 0) ctrl = POWER_CONTROL_ON;
        else if (strcmp(argv[1], "auto") == 0) ctrl = POWER_CONTROL_AUTO;
        else if (strcmp(argv[1], "suspend") == 0) ctrl = POWER_CONTROL_SUSPEND;
        else {
            fprintf(stderr, "Invalid control mode: %s\n", argv[1]);
            exit(1);
        }
        
        if (usb_set_power_control(path, ctrl) == 0)
            printf("Power control set to %s\n", argv[1]);
        else
            fprintf(stderr, "Failed to set power control (try running as root)\n");
            
    } else if (strcmp(argv[0], "wakeup") == 0 && argc >= 2) {
        gboolean enable = (strcmp(argv[1], "enable") == 0);
        if (usb_set_wakeup_enabled(path, enable) == 0)
            printf("Wake-up %s\n", enable ? "enabled" : "disabled");
        else
            fprintf(stderr, "Failed to set wake-up\n");
            
    } else if (strcmp(argv[0], "delay") == 0 && argc >= 2) {
        int delay = atoi(argv[1]);
        if (usb_set_autosuspend_delay(path, delay) == 0)
            printf("Autosuspend delay set to %d ms\n", delay);
        else
            fprintf(stderr, "Failed to set autosuspend delay\n");
    } else {
        fprintf(stderr, "Unknown power subcommand: %s\n", argv[0]);
        exit(1);
    }
}

static void cmd_authorize(int bus, int dev, const char *action)
{
    char *path = find_device_path(bus, dev);
    if (!path) {
        fprintf(stderr, "Device %d:%d not found\n", bus, dev);
        exit(1);
    }
    
    gboolean auth;
    if (strcmp(action, "allow") == 0 || strcmp(action, "yes") == 0)
        auth = TRUE;
    else if (strcmp(action, "deny") == 0 || strcmp(action, "no") == 0)
        auth = FALSE;
    else {
        fprintf(stderr, "Invalid action: %s (use 'allow' or 'deny')\n", action);
        exit(1);
    }
    
    if (usb_set_authorization(path, auth) == 0)
        printf("Device %d:%d %s\n", bus, dev, auth ? "authorized" : "deauthorized");
    else
        fprintf(stderr, "Failed to change authorization (try running as root)\n");
}

int main(int argc, char **argv)
{
    if (argc < 2) {
        print_usage();
        return 1;
    }
    
    if (strcmp(argv[1], "-h") == 0 || strcmp(argv[1], "--help") == 0) {
        print_usage();
        return 0;
    }
    
    if (strcmp(argv[1], "list") == 0) {
        cmd_list();
        return 0;
    }
    
    if (strcmp(argv[1], "show") == 0 && argc >= 3) {
        int bus, dev;
        if (sscanf(argv[2], "%d:%d", &bus, &dev) != 2) {
            fprintf(stderr, "Invalid device specification. Use bus:dev format (e.g., 1:5)\n");
            return 1;
        }
        cmd_show(bus, dev);
        return 0;
    }
    
    if (strcmp(argv[1], "power") == 0 && argc >= 4) {
        int bus, dev;
        if (sscanf(argv[2], "%d:%d", &bus, &dev) != 2) {
            fprintf(stderr, "Invalid device specification\n");
            return 1;
        }
        cmd_power(bus, dev, argc - 3, &argv[3]);
        return 0;
    }
    
    if (strcmp(argv[1], "authorize") == 0 && argc >= 4) {
        int bus, dev;
        if (sscanf(argv[2], "%d:%d", &bus, &dev) != 2) {
            fprintf(stderr, "Invalid device specification\n");
            return 1;
        }
        cmd_authorize(bus, dev, argv[3]);
        return 0;
    }
    
    fprintf(stderr, "Unknown command: %s\n", argv[1]);
    print_usage();
    return 1;
}
```

### Step 9: Update Makefile.am for CLI Tool

Add to **Makefile.am**:

```makefile
bin_PROGRAMS = usbview usbviewctl

usbviewctl_SOURCES = cli/usbviewctl.c power_manager.c power_manager.h
usbviewctl_LDADD = $(GLIB_LIBS)
```

---

## Testing Your Changes

After implementing these changes:

```bash
# Build the project
./autogen.sh
./configure
make

# Test the CLI tool
sudo ./usbviewctl list
sudo ./usbviewctl show 1:1
sudo ./usbviewctl power 1:1 status

# Run the GUI
./usbview
```

---

## Next Steps

After completing Phase 1-4, you can proceed with:

1. **Security Manager** - Implement device authorization policies
2. **Event System** - Add device connect/disconnect notifications
3. **Configuration Management** - Allow switching between device configurations
4. **Plugin Architecture** - Enable third-party extensions
5. **GTK4 Migration** - Modernize the UI framework

Each phase builds on the previous one, creating a comprehensive enterprise-grade USB management solution.
