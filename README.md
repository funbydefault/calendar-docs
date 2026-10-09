# Calendar 1.0

Calendar gives an Unreal Engine game configurable dates, time advancement, and scheduled events. Configure a **Calendar Definition** asset, add a **Calendar Component** to an actor, and connect its Blueprint nodes and events to your game.

> [!NOTE]
> This version is in release preparation. Engine and platform compatibility, multiplayer acceptance, and the final customer package are still being verified.

## Contents

- [Quickstart](#quickstart)
- [Calendar rules](#calendar-rules)
- [Date operations](#date-operations)
- [Automatic time advancement](#automatic-time-advancement)
- [Calendar events](#calendar-events)
- [Save and restore](#save-and-restore)
- [Authority and networking](#authority-and-networking)
- [Validation and troubleshooting](#validation-and-troubleshooting)
- [Blueprint reference](#blueprint-reference)
- [Support](#support)

## Quickstart

You can set up and use Calendar entirely in the Unreal Editor and Blueprints.

1. Install the plugin, enable **Calendar** in **Edit > Plugins**, and restart the editor if prompted.
2. In your project's Content Browser, create a **Data Asset** and choose **Calendar Definition** as its type. Save it in your own project, for example as `DA_MyCalendar`.
3. Open the asset and set **Definition ID** to a stable name such as `MyCalendar`.
4. Add one **Months** entry: ID `MonthOne`, Display Name `Month One`, and Day Count `30`. Add one **Weekdays** entry: ID `DayOne` and Display Name `Day One`. Leave **Epoch Weekday Index** at `0`, the clock at `24 / 60 / 60`, and **Start Date** at year `1`, month `1`, day `1`, time `00:00:00`.
5. Create or open an Actor Blueprint. Use **Add Component** to add **Calendar Component**, then assign `DA_MyCalendar` to its **Definition**. Keep **Initialize on Begin Play** enabled.
6. For a visible first test, set **Auto Advance Unit** to `Day`, **Auto Advance Amount** to `1`, and **Auto Advance Interval Seconds** to `1`. Enable **Start Auto Advance on Begin Play**.
7. In the actor's Event Graph, bind **On Calendar Changed** on the Calendar Component to a custom event. Pass the event's Current Date and your definition to **Format Calendar Date**, then print or display the returned text. Read the component's Current Date once on Begin Play to display the initial date as well.
8. Place the actor in your own level and play. The displayed calendar should move forward one day per game-time second. Stop play, change the definition or time settings, and repeat to try your own rules.

> [!TIP]
> Use the component's **Current Date** and **Format Calendar Date** in your own widget. Calendar provides the date and events; your game controls their presentation and input mappings.

## Calendar rules

Create separate definition assets for calendars with different rules. Treat the assigned definition as fixed during a play session. After changing rules, validate them and initialize the component again.

| Setting | Meaning |
|---|---|
| Definition ID | Stable identity used by snapshots and diagnostics. Keep it unique and preserve it when updating an existing calendar. |
| Months | Ordered months. Each needs a unique ID and at least one day. The first entry is month `1`. |
| Weekdays | Ordered weekday cycle. Each needs a unique ID. Its entry count defines the length of a week. |
| Epoch Weekday Index | Zero-based weekday assigned to year `1`, month `1`, day `1`. It must reference a weekday entry. |
| Clock | Hours per day, minutes per hour, and seconds per minute. Each value must be at least `1`. |
| Start Date | Date used when initializing or resetting the component. It must fit the calendar rules. |
| Year Adjustment Rules | Add or remove days from a selected month in repeating years. |
| Event Categories | Ordered groups of scheduled event entries. |

Years, months, and days start at `1`. Hours, minutes, and seconds start at `0`. Dates before year `1` and values outside the definition's ranges are rejected.

A year adjustment uses **First Year**, **Interval Years**, **Month**, and a signed **Day Adjustment**. For example, first year `4`, interval `4`, month `2`, and adjustment `1` adds one day to month `2` every fourth year from year `4`. Several matching rules combine. Validate the definition after editing them; every resulting month must remain valid.

**Season ID** on a month is an optional label for your game. Calendar does not create weather or seasonal effects.

## Date operations

Use a reference to your actor's Calendar Component to call these nodes:

| Node | Use |
|---|---|
| Initialize Calendar | Validate the definition and copy its Start Date into the component. An optional replacement definition may be supplied. |
| Try Set Date | Set an exact valid destination date. Invalid values are rejected. |
| Try Advance | Move forward or backward by a signed number of seconds, minutes, hours, days, weeks, months, or years. |
| Try Step | Apply the configured automatic unit and amount once, without starting a timer. |
| Reset to Start Date | Return to the definition's Start Date. Event evaluation is off by default for this reset. |

Each returns a **Calendar Operation Result**. Break the result and check **Success** before using the new state. On failure, read **Message**, **Context**, and **Suggested Action**.

For month and year moves, choose the missing-day policy:

- **Clamp Day:** use the destination month's last day when the original day is missing.
- **Reject:** leave the date unchanged and return a failure.

For example, advancing from day `31` to a month with `30` days gives day `30` with Clamp Day. Reject keeps the original date. An amount of `0` is a successful no-op.

The component exposes **On Calendar Changed**, **On Day Changed**, **On Month Changed**, and **On Year Changed**. Bind these delegates in Blueprints to update your UI or gameplay. A jump produces notifications for the committed change, not a notification for every intermediate date.

## Automatic time advancement

Automatic advancement uses a world timer. The Calendar Component does not need Tick.

| Setting | Default | Meaning |
|---|---:|---|
| Start Auto Advance on Begin Play | Off | Start the timer after successful automatic initialization. |
| Auto Advance Unit | Second | Calendar unit moved by each timer callback. |
| Auto Advance Amount | `1` | Signed non-zero number of units moved per callback. |
| Auto Advance Interval Seconds | `1` | Game-time interval between callbacks. It must be finite and at least `0.001` seconds. |
| Auto Advance Adjustment Policy | Clamp Day | Missing-day behavior for month and year moves. |

Use **Start Auto Advance** to start or resume, **Pause Auto Advance** to pause a running timer, and **Stop Auto Advance** to clear it. **Configure Auto Advance** changes the settings while the timer is stopped. Stop it before reconfiguring, then start it again.

**On Auto Advance State Changed** reports Stopped, Running, or Paused. World pause and time dilation affect this game-time timer. A failed automatic step stops the timer; fix the reported problem before restarting it.

## Calendar events

Add an **Event Category** with a unique ID to your definition, then add event entries. Each entry needs an ID unique across the entire definition and a valid **First Occurrence**.

For a one-time entry, leave **Repeats** off. For a repeating entry, enable it and choose **Repeat Unit** and a positive **Repeat Interval**. **Enabled** controls whether an entry participates in evaluation.

Bind the component's **On Event Triggered** delegate and inspect its context. It includes the event and category identity, scheduled occurrence, previous/current dates, and reason for the change. You can use the IDs to connect entries to your own game logic.

For reusable custom behavior, create a Blueprint derived from **Calendar Event**, implement **Trigger Calendar Event**, and assign that Blueprint class as the entry's **Handler Class**. The handler is cached for that category/event pair. Its object state is not included in calendar save snapshots.

> [!IMPORTANT]
> Events match the exact destination date and time. Moving past an occurrence does not replay it. A monthly event keeps its original day and time; it does not clamp to a shorter month's final day. Explicitly evaluating the same matching date again can trigger it again. Store any once-only gameplay completion state in your game.

**Evaluate Events at Current Date** evaluates without changing the date. On date-changing nodes, the **Evaluate Events** option controls whether matching entries are dispatched. Handler execution and On Event Triggered occur in definition order. Avoid changing the same calendar from inside its event handler; nested calendar mutations are rejected.

**Max Events Per Evaluation** defaults to `256`, with a supported range of `1` to `10000`. If a destination matches more events than the limit, evaluation fails safely. Reduce the simultaneous entries or intentionally increase the limit after reviewing the cost.

## Save and restore

Calendar supplies snapshot data. Your game's SaveGame object, save slots, storage, and loading flow remain yours.

1. Create a Blueprint class derived from Unreal's **SaveGame** and add a variable of type **Calendar Save Snapshot**.
2. Call **Try Create Snapshot** on an initialized Calendar Component. On Success, put Out Snapshot into that variable and use your normal **Save Game to Slot** flow.
3. Load the SaveGame object. Assign and initialize the same definition on the target component, then call **Restore Snapshot** with the saved value.
4. Check the returned result before continuing your load flow.

A snapshot includes its schema, Definition ID, date, and automatic-advance settings/state. It excludes event-handler object state. Restore rejects incompatible schemas, different Definition IDs, invalid dates, or invalid timer settings.

**Restore Auto Advance** controls whether a saved running or paused timer is restored. Turning it off still restores timer settings but leaves the timer stopped. Restoration does not replay scheduled events.

> [!WARNING]
> Keep Definition IDs stable. Changing month order, lengths, or other rules can make existing saved dates invalid or change their meaning. Plan and test your own save migration before changing a calendar used by released saves.

## Authority and networking

Replication is off by default. A non-replicated Calendar Component is an independent local timeline.

For a shared timeline, use a server-owned replicated actor, enable replication on both the actor and Calendar Component, and assign the same installed definition assets to the participating builds. Definition and Current Date are the replicated state. Date-change delegates can update each client's local display when new dates arrive.

Perform calendar mutations, automatic stepping, scheduled event evaluation, and snapshot creation/restoration on the server authority. The plugin rejects those requests on a replicated non-authority component. It does not provide a client-to-server request RPC. Route player requests through your game's owned, validated server request flow.

Scheduled handlers and On Event Triggered execute on the authority. Client UI, timer lifecycle settings, and event-handler state do not become shared simply because Current Date is replicated. Replicate your own resulting gameplay state where needed. Keep local input and UI on their owning client.

> [!NOTE]
> Listen-server, dedicated-server, late-join, reconnect, ownership-change, latency/loss, and separate packaged-process acceptance are pending for this candidate. These implementation notes do not establish a verified multiplayer support claim.

## Validation and troubleshooting

Use **Validate Calendar Definition** to obtain all definition issues, or **Validate Configuration** on the component to inspect its assigned rules and settings. Each issue includes a message, context, and suggested action.

| Symptom | Check |
|---|---|
| Calendar is not initialized | Assign a valid definition, enable Initialize on Begin Play, or call Initialize Calendar. |
| Date change is rejected | Check the operation result, valid date ranges, missing-day policy, and authority. |
| Timer does not advance | Initialize first, use a non-zero amount and valid interval, then start it. Check world pause and the last failure. |
| An event is missing | Check Enabled, its exact date/time anchor, repeat interval, destination-only behavior, and authority. |
| Snapshot cannot restore | Check schema, Definition ID, date validity, timer settings, and initialization. |
| Replicated calendar cannot initialize | Enable replication on its owner as well as the component, and initialize on the server. |

**Get Diagnostic Snapshot** returns current date, definition identity, initialization, timer, replication/authority, event-count, and last-operation state. Bind **On Operation Failed** for concise error feedback in your own tools. **Enable Debug Logging** is off by default. **Dump Diagnostics to Log** writes one bounded diagnostic entry and is a development-only node.

For support, include the operation result and a small reproducible case. Remove sensitive project information before sharing logs.

## Blueprint reference

These utility nodes calculate or inspect values without changing a component's date:

| Node | Result |
|---|---|
| Is Calendar Date Valid | Test an exact date against the definition. |
| Try Advance Calendar Date | Calculate a destination using the chosen unit, amount, and missing-day policy. |
| Try Get Calendar Day Index | Convert a valid date to a zero-based whole-day index from `1/1/1`. |
| Try Get Elapsed Calendar Days | Signed difference between two day indices; time-of-day fields do not affect the difference. |
| Try Get Calendar Weekday | Resolve a weekday index and entry. |
| Does Calendar Event Occur | Match an event at an exact candidate date, without running its handler. |
| Format Calendar Date | Create display text using the definition's month names. Formatting does not validate the date. |

On the definition itself, **Get Days in Month**, **Validate Definition**, and **Is Date Valid** provide rule lookup and validation. Use the native Blueprint types **Calendar Date Time**, **Calendar Operation Result**, **Calendar Validation Report**, **Calendar Event Context**, **Calendar Save Snapshot**, and **Calendar Diagnostic Snapshot** to connect your logic.

## Support

Contact **FunByDefault** at [funbydefault.dev@gmail.com](mailto:funbydefault.dev@gmail.com). Include Calendar version, Unreal Engine version, platform, relevant operation result, and concise steps to reproduce the problem in a clean project. Do not send credentials or post private project files publicly.
