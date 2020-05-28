###### tags: `R4MP`

# Design Proposal

## Some pontifications

First, some assumption and suppositions.

### On mobile

R4MP is required to work in mobile and desktop pages equally well without dropping features in either mode. This seems like a simple problem, since it's not possible to tell if the page is running in a mobile browser or not. Multiple articles on the Internet suggest that it shouldn't even be tried at all since there are so many possible cases (big phone, small tablet, small monitor, a touch-screen laptop, a desktop with a touch-pad, etc.). Sniffing browser agent or availability of touch event is a fool's errand it seems.

Instead, the UI layout will depend on the size of the R4MP container. So, technically, embedding a 320x640 R4MP instance into a page and viewing it on a desktop will display "mobile" layout.

Most of the UI frameworks (Bootstrap, Bulma, Vanilla Framework, etc.) rely on screen breakpoints to provide different styles to components and adjust font size. Not only such behaviour won't suit an embedded R4MP instance, it will mess up the host page layout.

[Tailwind CSS](https://tailwindcss.com) is a low-level utility CSS framework which provides multitude helper classes but doesn't not define any component. It's build with PostCSS and there is some amount of flexibility in how the framework is set up. Using with [Responsive Components](https://philipwalton.com/articles/responsive-components-a-solution-to-the-container-queries-problem/) approach, all the CSS utility functions and helpers can be scoped under pseudo-breakpoints which let component layouts react to the size of their top-level container (R4MP instance).

There will be four pseudo-breakpoints: **xs**, **sm**, **md**, **lg** (**xs** and **sm** will have smaller font and button sizes).

They roughly correspond to the following use cases:

![](https://i.imgur.com/IOdDaI4.png)

*Fig. 1: Full-width instance or full-screen mode on a large screen*

![](https://i.imgur.com/8zYGOEG.png)

*Fig. 2: Medium screen or embedded into a larger page*

![](https://i.imgur.com/zORXBc3.png)

*Fig. 3: Smallish screen or tablet or embedded into a page*

![](https://i.imgur.com/sZxA0OS.png)

*Fig. 4: Mobile screen or embedded into a page*

*To recap:* it's possible to have a **xs** layout embedded into a desktop-sized page and a **lg** layout on a chunky tablet.

### On panels

Panels provide place for most of the R4MP UI controls.

Relevant R4MP constraints:

- needs to work reasonably well on mobile and desktop
- allows for third-party modules which can add arbitrary UI elements to the screen
- behaves consistently across all layouts, so users' expectation are not thwarted
- be pleasant to use, if possible
- be easy to code (very important)

Several previous attempts at tackling this problem led to development of a flexible grid where panels can be added at arbitrary points on top of the map with underlying logic that control which panels were visible (panels could be closable and permanent, with closable panels able to overlay permanent ones). This approach works, but only if the screen size stay relatively the same, as panel positions become really hard to figure out as the container shirks. As the results, all the extra panels were positioned in the left upper corner over the main panel anyway.

To tackle this problem, the following is proposed:

- all panels take the full height of the R4MP container, although their width can be specified (min/max)
- panels (aka blades) have views/screens/pages/terminology tbd. (aka routes) and are stacked horizontally 
- on-map UI components (like CCCS date slider) are not panels

*Notes:*
- a `core panel` is anything that can be opened from the appbar at any time (search, legend, basemaps)
- a `non-core panel` or `transient panel` is something that opens as a response to a action, like the Details or Coordinate information panel. Such panels can be pinned/minimized/docked to the appbar to preserve their content for later. 

Also, no pop-up dialogs.

All panels will have a header with an obligatory title and some optional UI controls:

![](https://i.imgur.com/X8hNCRq.png)

*Fig. 5: Four-panel stack with the Details panel closed*

- back button in front of the title will switch to the previous panel screen (or close the panel in **xs** layout if there is no more screens to go back to)
- close and maximize buttons are only available on the >= **sm** layouts
- minimize / pin to appbar button is used for non-core panels which cannot be open directly from the appbar (the Details panel, for example; when pinned, it can be reopened from the appbar preserving the results)
- close button is used for non-core panels (like the Details panel) to fully close them, without preserving the results
- no more than three buttons should be displayed on the right side of the header with all extra controls should be hidden under a menu: four buttons total--three action buttons and the three-dots menu button;
- if there are many functions available in a panel, only the most important ones should be put in the panel header; the rest should go into the panel body;

#### Minimizing/pinning/docking a panel

When a transient panel is opened, a new button is added to the appbar to represent this panel. When the panel is closed, the button is removed. If the panel is minimized (or hidden by the panel stack logic), the button remains and clicking it will open the panel again.

*Note:*
- when a minimized panel is re-opened, it is added to top of the panel stack, not in its previous position; this is done to reduce the number of different panel behaviours.

![](https://i.imgur.com/AG4BRQb.png)

*Fig. 5.1: Non-core Details panel is opened and minimized*

#### All panels are equally tall

To avoid unnecessary logic in panel managements, all panels take up all the available vertical space in the R4MP container, in all layouts. Any and all UI controls can go into a panel as long as they fit (or use a scrollbar).

The modules that create panels will be able to specify min and/or max width of any panel; a panel can be also maximized to take up all space in the instance container (sort of like a large pop-up dialog).

The half-grid of the current RAMP implementation won't be possible anymore. However, a grid can have the max width set to 50% of the container creating a horizontal half-grid (when other panels are closed).

![](https://i.imgur.com/swEcib5.png)

*Fig. 5.2: Horizontal half-grid panel*

*Anecdotal supporting evidence:* Web pages on desktop usually* present information in a landscape layout which will leave more map visible with the horizontal half-grid than with vertical one.

*Concerns:* Having all panels take the maximum height is going to waste a lot of space and create too much white space for extremely simple datasets that return minimal data. Additionally, some clients might find this setup restrictive.

The main argument for full-height panel is layout control. Since R4MP is easily extensible by fixtures (modules), panels that are created by third-party code need to be managed in a predictable and consistent way. Letting panels be placed anywhere inside the container if liable to conflicts where panels overlap each other, and letting the panels be of arbitrary height, will create inconsistent layout where it's possible to have a tall panel, a (very) short panel, and another tall panel, creating a jagged bottom edge of the layer stack.

That said, if required, variable panel height can be implemented (for medium and large) layouts at a later point.

#### Panels have screens and are stacked horizontally

Each panel will have one or more screens it can display (a panel is a container of a certain size, and screen are different, but related UI components which can be displayed inside the panel). These are technically routes. In the current RAMP implementation, a single panel always displays the same content, which is simple but is cumbersome when to display related content a new panel needs to be created.

A panel should only contain screens that are relevant or related to each other. The Legend panel, for example, will have three screens: the layer list, layer settings, and layer metadata.

![](https://i.imgur.com/D5zuNil.png)

*Fig. 6: The Legend panel with three screens*

The legend panel does not contain a screen for layer data or feature details, although they both are related. The Datatable requires its own panel due to complexity of it, and the Details panel is separate, so it's possible to open it and the Datatable (or Legend) panel at the same time (on larger layouts at least).

In the example above, The Setting screen is part of the Legend panel. This approach makes behaviour consistent cross small/large layouts and conserves space on larger layouts (no need to close datatable when opening settings, etc.)

It should be also possible to (programmatically) open any panel with any pre-defined screen. For example: opening the layer Setting screen directly without showing the Layer list screen first.

#### Panel stack example

When more than one panel is open:

- panels stack horizontally, aligned to the left,
- no more than three panels are visible at the same time (deepening on the layout, it could be two or one; these numbers can also be tied to the config file or determined programmatically);
- when a fourth panel is open, the first panel on the left is hidden, but not fully closed
- when a panel is closed, a last panel to be hidden (if exists) is brought back
- when a panel is maximized, it doesn't change its position in the stack, but is visually made to cover all other panels
- since panel width can be configured by map authors, if there is not enough room for a new panel, the left-most panels are hidden until there is enough room;

![](https://i.imgur.com/ficiw9N.png)

*Fig. 7: Five-panel stack with three visible*

![](https://i.imgur.com/3tbf2D1.png)

*Fig. 8: Five-panel stack with the Details panel maximized*

![](https://i.imgur.com/7I6PCsK.png)

*Fig. 9: Four-panel stack with the Details panel closed*

On **xs** layout, panels are also stacked on top of each other since there is no space to display more than one at a time.

![](https://i.imgur.com/DRdE6gb.png)

*Fig. 10: XS layout with stacked panels*

On **xs** layout stack:

- there is no close button in the panel header, only back button (the user needs to close panels in the reverse order to how they were opened)
- panels cannot be maximized and always have the same size
- *panel-peek* effect will be available to panels 

*Notes:*

- The removal of the close button reflects differences in navigation on desktop vs mobile layouts.
- On larger layouts, it's possible to close the second panel in a stack of three; on small layout, it's required to first close the third one and only then close the second one. So, on small layouts the user always goes back through the screens and panels. Closing and going back means the same thing on small layouts, and two different things on desktop.

#### Pinned panels

A situation is possible where there is an active tool in a panel that the user always wants to be on the screen, and using this tool generates a number of new transient information panels. These new panels will eventually push the tool panel out of the visible part of the panel stack and likely frustrate the user.

Such a panel can be pinned, so it "always" remains in view unless there is no more room to open a new panel.

##### Only one pinned panel

In the initial implementation, the user will only be able to pin a single panel. Pin controls will either be disabled on other panels when another panel is pinned, or pinning a new panel will automatically unpin the first one. TBD.

If this implementation proves troublesome (angry clients, death threats, etc.), an option to allow multiple panels to be pinned at once will be explored.

The pin option will be disabled by default on all panels and needs to be manually enabled by the map authors.

##### Overflow behaviour

When a new panel must be opened together with pinned panel, but there is no space, the pinned panel will be hidden, as any other panel. However, pinned panel will be first to restore as the space becomes available. 

This can lead to the pinned panel changing its position in the panel stack:

![](https://i.imgur.com/XQ1GWdW.png)

*Fig. 10.1: The Legend panel is pinned*

![](https://i.imgur.com/3sVAafL.png)

*Fig. 10.2: The Datatable is opened forcing both panels to be hidden*

![](https://i.imgur.com/UK2AxmZ.png)

*Fig. 10.3: When the Datatable is closed, the Legend panel is first to be restored*

In the above example, the Legend panel switches places with the Random plugin panel because the pinned panel is always restored first.

###### Enhancement

As a potential enhancement, the "slide-and-hide" behaviour will be implemented for the pinned panel instead of completely hiding it.

![](https://i.imgur.com/fTU58S7.png)

*Fig. 10.4: Slide-and-hide for a pinned panel*

Where there is no more space for a new panel, a pinned panel slides under the appbar on the left that only a sliver of it remains in view. The new panel is opened in the freed-up space.

When the user hovers (or tabs) to the pinned panel, the panel slides out (and is interactive) and partially covers the other opened panel. It slides back under the appbar when the cursor moves away.

#### On-map UI components

Any other UI components that don't fit into a panel, like CSSS date slider, or an overview map, or something like that, will be placed directly above the map, and subject to be covered by any panel. 

Positioning of such elements on all layouts is the responsibility of the module that created it, or the host page (if, for example, a host page imports two different slider controls that want to occupy the same space).  How this positioning is to be done, is tbd.

#### Side/hamburger menu 

The R4MP sidemenu is opened through the hamburger/gears button in the lower left corner of the app. On small layouts, it can open as a slide-out panel as in the current RAMP; on larger layouts, as a drop-down menu. TBD.

## Layout examples

### Identify query / Details panel

The Identify query panel will have three screens:

- a list of layers and the number of results per layer
- a list of results for a particular layer (displayed after clicking on that layer in the previous screen)
- result details (displayed after clicking on that result in the previous screen)

![](https://i.imgur.com/se1ks1S.png)

*Fig. 11: Identify query panel with three screens*

### Three-panel lg layout

Here's an example of how three different panels might be displayed on a **lg** layout.

![](https://i.imgur.com/VAr8Q9u.png)

*Fig. 12: LG instance with the Legend panel open*

Since the Datatable panel has its max width set to 100%, it takes all available space:

![](https://i.imgur.com/rZddCfN.png)

*Fig. 13: LG instance with the Legend and Datatable panels open*

Clicking on a feature in the data table, opens the Details panel. Unlike in the current RAMP implementation, the Details panel is open on the right, and the Datatable panel is reduced in width up-to its min width (if there is not enough space to fit min-width of the Datatable panel and the Details panel, the Legend panel will be hidden).

![](https://i.imgur.com/e9xDm2r.png)

*Fig. 14: LG instance with the Legend, Datatable, and Details panels open*

### Two-panel md layout

![](https://i.imgur.com/RIxRs6A.png)

*Fig. 16: MD instance with the Legend and Datatable panels open*

![](https://i.imgur.com/gK6XFoj.png)

*Fig. 17: MD instance with the Legend (hidden), Datatable, and Details panels open*

### Three-panel xs layout

On **xs** layout, panels always overlay each other and the map. Opening the Legend panel hides the map, opening the Datatable panel overlays the Legend panel, etc. 

![](https://i.imgur.com/k68d5c8.png)

*Fig. 18: MD instance with the Legend (left), Legend and Datatable (middle), and Legend, Datatable and Details (right) panels open*

### "Panel-peek" on xs layout

On **xs** layout, since the panel always cover all of the map, there will be an option to display a panel-popup to notify the user that a panel is available to be opened. This can be used when opening the Identify results/Details panel in response to user clicking/tapping on the map. Any module will be able to use this option with any panel.

![](https://i.imgur.com/Aw3YSrY.png)

*Fig. 19: XS layout with no panels open*

Clicking or tapping on the map set an identify marker to that stop and opens a floating popup (similar to a toast notification, but not a toast) with the name of the panel and an option line of information.

![](https://i.imgur.com/MudWkav.png)

*Fig. 20: XS layout with a identify marker and a Details panel-popup*

Tapping or clicking on this popup will open the full Details panel. The popup has a close button to dismiss the popup. Opening any other panel will also dismiss the popup.

![](https://i.imgur.com/dJd45NK.png)

*Fig. 21: XS layout with the Details panel open*

### Pinned panels

![](https://i.imgur.com/XKcOsE3.png)

*Fig. 22: The Legend panel is pinned*

![](https://i.imgur.com/WP9Tttb.png)

*Fig. 23: A new panel is opened and forces the Datatable panel to be hidden*

![](https://i.imgur.com/x2ae3qd.png)

*Fig. 24: The Details and Help panels are closed and the Datatable is restored*