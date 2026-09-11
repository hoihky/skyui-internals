---
title: SkyUI Internals
order: 0
---

# SkyUI Internals: Building Custom Avalonia Controls

A comprehensive developer guide to the SkyUI toolkit for Avalonia 12 — architecture, design patterns, Avalonia concepts, and detailed implementation walkthroughs for **30+ controls**.

## What You Will Learn

- The Avalonia property system, visual and logical trees, templates, bindings, and routed events
- SkyUI package architecture, design tokens, and theme authoring
- Step-by-step recipes for building custom controls from scratch
- In-depth implementation of layout, form, list, data, navigation, shell, feedback, menu, primitive, mobile, and advanced controls
- Recurring patterns: adapters, host facades, abstract bases, attached properties, pseudo-classes, custom panels

## Complete Chapter Guide

### Part I — Foundations
1. The Avalonia Control Model
2. Package Architecture
3. Design Tokens and Theming
4. The Control-Building Recipe

### Part II — Layout
5. SkyCard and SkyDivider
6. Responsive Layout (SkyResponsiveGrid, SkyGridLayout)
7. SkyExpander and SkyLoadingOverlay

### Part III — Forms and Input
8. SkyFormField
9. SkyAutocomplete and Validation
10. SkyRadioGroup, SkySlider, and SkyNumericUpDown
11. Date and Time Pickers (SkyDatePicker, SkyDateRangePicker)

### Part IV — Lists and Trees
12. CheckedListBox
13. SkyVirtualTreeView and SkyAccordion

### Part V — Data Presentation
14. Pagination and KPI Tiles
15. SkyVirtualDataGrid
16. FilterEditor

### Part VI — Navigation and Shell
17. SkyNavigationView
18. SkySplitView and Page Scaffolds
19. SkyTabView and SkyBreadcrumb
20. SkyCommandBar, SkyPageHeader, and SkyDrawer

### Part VII — Feedback and Overlays
21. Dialog and Snackbar Hosts
22. SkyAlert, SkyProgressRing, and SkySkeleton
23. SkyMenuBar and SkyContextMenu

### Part VIII — Primitives
24. Chip, Badge, and Avatar
25. Attached Properties
26. SkyIcon and SkyFontIcon

### Part IX — Advanced
27. DiagramSurface
28. Mobile Primitives (SkySafeArea, SkyActionSheet)
29. VideoTimeline

## Controls Covered

| Category | Controls |
|----------|----------|
| Layout | SkyCard, SkyDivider, SkyResponsiveGrid, SkyGridLayout, SkyExpander, SkyLoadingOverlay |
| Forms | SkyFormField, SkyAutocomplete, SkyRadioGroup, SkySlider, SkyNumericUpDown, SkyComboBoxField, SkyMaskedTextBox, SkySearchBox, SkyPasswordBox |
| Pickers | SkyDatePicker, SkyTimePicker, SkyCalendar, SkyDateRangePicker, SkyFilePicker |
| Lists | CheckedListBox, SkyVirtualTreeView, SkyAccordion |
| Data | SkyPagination, SkyDataPager, SkyKpiTile, SkyVirtualDataGrid, FilterEditor |
| Navigation | SkyNavigationView, SkyTabView, SkyBreadcrumb |
| Shell | SkySplitView, SkyListDetailPage, SkyCommandBar, SkyPageHeader, SkyDrawer, SkyStatusBar |
| Feedback | SkyDialogHost, SkySnackbarHost, SkyMessageBox, SkyAlert, SkyBanner, SkyProgressRing, SkySkeleton, SkySheetHost |
| Menus | SkyMenuBar, SkyContextMenu, SkyMenuFlyout |
| Primitives | Chip, Badge, Avatar, SkyIcon, SkyButtonProperties, SkyTooltip |
| Mobile | SkySafeArea, SkyKeyboardInset, SkyActionSheet, SkyTouchTarget |
| Advanced | DiagramSurface, VideoTimeline |

## Reading Order

Parts I–IV are essential for learning custom control development. Parts V–IX can be read selectively based on your application needs.

## Conventions

- Code samples reflect real SkyUI patterns
- Template parts use `PART_*` naming
- Style classes use the `sky-*` prefix
- Repository paths are relative (e.g. `src/SkyUI/Controls/...`)

Begin with **Part I, Chapter 1 — The Avalonia Control Model**.
