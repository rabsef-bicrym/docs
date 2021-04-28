+++
title = "6. Things Tudu and People TuSee"
weight = 7
template = "doc.html"
+++

In this last part of our guide, you'll learn how to add networking to our app. You'll be able to subscribe to your friends or coworkers task lists and even, if they permit you, edit those lists (with the same level of control you have over your own lists). As a proprietor of a task list, you'll also be able to select, of those people who subscribe to your task list, who can edit and who is precluded from editing but may still look. You can even kick them from their subscription, if you're really annoyed.

This lesson includes a lot of changes but few of them are structural and most of them are just iterations on themes of prior lessons - we hope you find it fairly easy to follow.

Let's get after it then.

## Learning Checklist {#learning-checklist}
* How to use wires and paths to send and receive data.
* How to add new mark files to your program for various purposes.

## Goals {#goals}
* Upgrade your %gall agent's state to handle multiple todo lists from various ships.
* Upgrade your Earth app to handle multiple todo lists, including selecting between various lists and subscribing to additional lists.
* Add new pokes to your agent that allow for specification of the ship to which those pokes should apply.
    * This is how you'll handle pokes to remote ships from our Earth app.
    * We'll also see how you can use the same pokes you've already written to poke remote ships through the dojo.
* Add new pokes to handle receipt of new tasks data from ships you subscribe to when they make changes.

## Prerequisites {#prerequisites}
* The Earth web app as modified in [the last part of this guide](./earth-to-mars-comms.md).
    * A copy of our current Earth web app can be found in [/src-lesson6/todomvc-start](https://github.com/rabsef-bicrym/tudumvc/tree/main/src-lesson6/todomvc-start).
* A new Fake Ship (probably ~zod) with whom you can share task!
* **NOTE:** We've included a copy of all the files you need for this lesson _in their completed form_ in the folder [/src-lesson6](https://github.com/rabsef-bicrym/tudumvc/tree/main/src-lesson6), but you should try doing this on your own instead of just copying our files in. No cheating!

## The Lesson (Part I) {#the-lesson-pt-I}
We're going to start by updating our available pokes to accommodate actions required to share our todo lists, updating our state to hold several todo lists owned by various ships and adding to the state a tuple of sets of editors; requested-editors, approved-editors and denied-editors.

Begin by launching your Fake Ship and a new Fake Ship (of your choice, we'll assume ~zod) with the Lesson 5 version of tudumvc installed and started. These two ships should be running on the same machine so they can discover eachother (fake ships are not networked _outside_ a given machine, but on the same machine they are inter-discoverable!)

You'll also want to have two terminals running to copy files from a central editing folder to _both_ of your fake ships simultaneously, to allow you to update both of them quickly.

### Updating the Types {#lesson-pt-I-types}
First, update the agent's /sur file to have two new types - `shared-tasks` and `editors`:

#### Adding Types
The changes to /sur should be familiar at this point, at least in terms of how we implement them:
<table>
<tr>
<td>
:: initial sur file version
</td>
<td>
:: new sur file version
</td>
</tr>
<tr>
<td>

```hoon
+$  tasks  (map id=@ud [label=@tU done=?])
```
</td>
<td>

```hoon
+$  tasks  (map id=@ud [label=@tU done=?])
+$  shared-tasks  (map owner=ship task-list=tasks)
+$  editors  [requested=(set ship) approved=(set ship) denied=(set ship)]
```
</td>
</tr>
</table>

As you can see, `shared-tasks` is simply a map of ships to the pre-existing type (`tasks`).

Editors is slightly more complex, but it works exactly like `+cors-registry` works. We're making a tuple of `requested` `approved` and `denied` editors (the names are descriptive of their capacities - only `approved` editors can edit your task lists), each of which are `(set ship)`s. A [set](https://urbit.org/docs/hoon/reference/stdlib/2o/#set) is just a mold that creates a list-like structure that _only_ allows for unique items to be added. By using `(set ship)` instead of `(list ship)` or some other structure, we can: (1) more easily work on our `(set ship)`s using [set logic](https://urbit.org/docs/hoon/reference/stdlib/2h/) and (2) ensure that nobody gets listed twice in any one category. Additionally, you'll abuild logic to make sure that any action taken on an editor removes them from their current category and places them in another category. That way, if you `%kick` someone from their subscription, they can't just re-subscribe and start editing your tasks. The uniqueness checks required for the above are further improved by using set logic.

Now to add some pokes to your `action` type in the same /sur file to accommodate subscription actions:

#### Adding to the `action` Type
Thinking through what you'll need here, you should prepare for the following actions:
* Subscribing to a ship's todo list.
    * Will need to specify a ship to which we will subscribe.
* Unsubscribing to a ship's todo list.
    * Will need to specify a ship to which we will subscribe.
* `%kick`ing a subscriber (or, forcefully removing someone who subscribes to you).
    * Will need to specify a ship to `%kick`.
    * Will also need to specify a path to `%kick` them from - we'll talk about paths more later, but these are basically the data feed on which they're receiving your todo list. Theoretically, since `%tudumvc` only has one data feed, [we could simply remove them from all possible paths](https://github.com/timlucmiptev/gall-guide/blob/master/poke.md#example-4-kicking-a-subscriber-from-the-host), but we'll still allow our poke to specify one or more paths, as a best practice.
* Allowing or denying a request from a subscriber to edit your todo list.
    * Will need to specify a `(list ship)`s that you want to edit (so you can mass approve or deny).
    * Will need to specify a status for those ships (`%approve` or `%deny`)

<table>
<tr>
<td>
:: initial sur file version
</td>
<td>
:: new sur file version
</td>
</tr>
<tr>
<td>

```hoon
+$  action
  $%
  [%add-task task=@tU]
  [%remove-task id=@ud]
  [%mark-complete id=@ud]
  [%edit-task id=@ud label=@tU]
  ==
```
</td>
<td>

```hoon
+$  action
  $%
    [%add-task label=@tU]
    [%remove-task id=@ud]
    [%mark-complete id=@ud]
    [%edit-task id=@ud label=@tU]
    :: New action types for handling subscriptions on urbit side
    ::
    [%sub partner=ship]
    [%unsub partner=ship]
    [%force-remove paths=(list path) partner=ship]
    [%edit partners=(list ship) status=?(%approve %deny)]
  ==
```
</td>
</tr>
</table>

These should all make relative sense at this point. If, for instance, you want to subscribe to a partner, you could just key into dojo `:tudumvc &tudumvc-action [%sub ~zod]`. Similarly, you might `:tudumvc &tudumvc-action [%unsub ~zod]` to cancel that subscription later. We'll later write handlers in the main agent file to handle all this, so don't try it just yet. Nonetheless, the intended use should be fairly clear.

Having subscribed to someone, we are going to need some way of: (1) receiving their full list of tasks when we first subscribe and (2) receiving incremental updates as their todo list changes over time. It would be fairly inconvenient and expensive to receive the whole list each time, so we'll take each one of those updates almost exactly like we take updates to our own list:

#### Creating an `updates` Type
Add a type called `updates` to `%tudumvc`'s /sur file. It's going to fairly closely mirror the existing `action` type with the addition of one tagged union called `%full-send` which will send _all_ tasks to a subscriber when they first subscribe. The primary difference between `action` and `updates` is that all incremental updates from other ships will come with an `id=@ud` identifier, to to avoid `id` dis-union between proprietor and subscribers on individual todos:

<table>
<tr>
<td>
:: adding update type to our sur file (wherever)
</td>
</tr>
<td>

```hoon
:: Here we're creating a structure arm called updates and we're re-creating the
:: task actions that might come in as updates from the ships to which we subscribe
::
+$  updates
  $%
    [%task-add id=@ud label=@tU]
    [%task-remove id=@ud]
    [%task-complete id=@ud done=?]
    [%task-edit id=@ud label=@tU]
    [%full-send =tasks]
  ==
```
</td>
</tr>
</table>

You might note, at this point, that these types don't include the source of the update (the ship name). You won't have to specify the ship sending any data, %gall handles that for you. Each incoming poke to your ship will be identified by `src.bowl` (available associated with any incoming poke or `+on-agent` action) which will be the `@p` of the ship sending you the data.

Only the owner of a given list will ever send you `updates` regarding their state - this further ensures parity amongst all subscribers/editors/the original host in terms of the status of the tasks and their individual `id`s. You'll never receive conflicting or out-of-order instructions from various parties (editors and the host).

**NOTE:** These poke structures could probably be added to `action` just as easily as separated out into their own type, but we're going to use separate types for clarity. When we add Earth web pokes to specify the receiving ship of a given update to the task list, we're going to do the same thing, again, for clarity and separation.

Now to create a new /mar file to handle the `updates` type you've just created and then you'll be ready to work on the main app:

#### Adding `/mar/tudumvc/update.hoon`
The /mar file for your `updates` type is very basic - it effectively just expects incoming data to be a noun and does no real work to convert it from other potential input types - this is ok and expected, as `updates` pokes will only _ever_ be generated on the Urbit side of things, between ships with active subscriptions. Again, since `updates` pokes will only _ever_ come from the owner of the list, the updates will always come in the same order that they occur to the main ship; your state will never get out-of-order messages, causing your state to diverge from the host's.

This guide doesn't go into depth on the various arms available in a standard /mar file, outside of the `+grab` arm which is used to convert from a non-given mark type to the given mark type, as in from a JSON data mark to an `action` data mark type to allow TodoMVC (Earth web app) to communicate with `%tudumvc` (%gall agent) in the previous part of this guide.

The /mar file for `updates` only needs to accept incominging like-mark data from other ships, so our `+grab` arm will simply expect a noun and will coerce it into the `updates:tudumvc` type that we just created.

<table>
<tr>
<td>
:: create a file called update.hoon in your /mar/tudumvc folder
</td>
</tr>
<td>

```hoon
::
::  tudumvc-update mar file
::
/-  tudumvc
=,  dejs:format
|_  upd=updates:tudumvc
++  grad  %noun
++  grow
  |%
  ++  noun  upd
  --
++  grab
  |%
  ++  noun  updates:tudumvc
  --
--
```
</td>
</tr>
</table>

Again, all this is really doing (of import) is the `+grab` arm is saying "if I receive a piece of data as a `noun`, coerce it into an `updates:tudumvc` type, generally by reading the head of the tagged union vase received.

Now to work on the agent, itself:

### Updating `/app/tudumvc.hoon` {#lesson-pt-I-state}
To start, you'll update the state of `%tudumvc` to accommodate multiple ships. After that, you'll need to update `+on-init`, `+on-load`, `+on-poke`, and `+on-watch`, as well as adding new features to `+on-agent` and your helper core:

#### The State
Update the state to include the new types that you've created (`shared-tasks` and `editors`):

<table>
<tr>
<td>
:: initial version (lines 3-18)
</td>
<td>
:: new version (lines 3-16)
</td>
</tr>
<tr>
<td>

```hoon
|%
+$  versioned-state
    $%  state-one
        state-zero
    ==
+$  state-zero
    $:  [%0 task=@tU]
    ==
+$  state-one
    $:  [%1 tasks=tasks:tudumvc]
    ==
+$  card  card:agent:gall
--
%-  agent:dbug
=|  state-one
=*  state  -
```
</td>
<td>

```hoon
|%
+$  versioned-state
    $%  state-two
        state-one
        state-zero
    ==
+$  state-zero  [%0 task=@tU]
+$  state-one   [%1 tasks=tasks:tudumvc]
+$  state-two   [%2 shared=shared-tasks:tudumvc editors=editors:tudumvc]
+$  card  card:agent:gall
--
%-  agent:dbug
=|  state-two
=*  state  -
```
</td>
</tr>
</table>

You're only really changing three things here:
1. Add state-two to the versioned state definition
2. Change the bunted state of the door to state-two (from state-one)
3. Make the state definitions a little more terse than they were previously - but these operate exactly the same:

<table>
<tr>
<td>
:: this
</td>
<td>
:: is equal to this
</td>
</tr>
<tr>
<td>

```hoon
+$  state-zero
    $:  [%0 task=@tU]
    ==
```
</td>
<td>

```hoon
+$  state-zero  [%0 task=@tU]
```
</td>
</tr>
</table>

This more terse definition lines things up nicely and presents the data with less white space - it doesn't impact the meaning of the lines at all.

Now for the changes to the arms of your %gall agent:

#### `++  on-init`
You only need to chage one line in `+ on-init` section - the one that populates the bunted `state-two` sample of the door:

<table>
<tr>
<td>
:: initial version (lines 26-32)
</td>
<td>
:: new version (lines 24-30)
</td>
</tr>
<tr>
<td>

```hoon
  ^-  (quip card _this)
  ~&  >  '%tudumvc app is online'
  =/  todo-react  [%file-server-action !>([%serve-dir /'~tudumvc' /app/tudumvc %.n %.n])]
  =.  state  [%1 `tasks:tudumvc`(~(put by tasks) 1 ['example task' %.n])]
  :_  this
  :~  [%pass /srv %agent [our.bowl %file-server] %poke todo-react]
  ==
```
</td>
<td>

```hoon
  ^-  (quip card _this)
  ~&  >  '%tudumvc app is online'
  =/  todo-react  [%file-server-action !>([%serve-dir /'~tudumvc' /app/tudumvc %.n %.n])]
  =.  state  [%2 `shared-tasks:tudumvc`(~(put by shared) our.bowl (my :~([1 ['example task' %.n]]))) [~ ~ ~]]
  :_  this
  :~  [%pass /srv %agent [our.bowl %file-server] %poke todo-react]
  ==
```
</td>
</tr>
</table>

Remember that `+on-init` is only run once, when the application is first started (as in not _upgraded_, but installed anew). The changed line does two things:

1. Adds to `shared-tasks` (a `(map ship tasks:tudumvc)`) a key of `our.bowl` (our ship name) and a value of that same initial task map your last `+on-init` arm  had. Basically you're nesting the prior state map construction (`tasks`) into a map of ships-to-`tasks` where each ship has its own version. We'll use this later to store the maps of `tasks` coming from those ships to which we subscribe.
2. Adds a new element to state; a tuple of `(set ship)`s that will store requested-, approved-, and denied-editors, starting off as empty sets (`[~ ~ ~]`).

With `+on-init` handled, let's make sure prior users of the app can upgrade to the latest version in `+on-load`:

#### `++  on-load`
Create an upgrade path for all prior versions of the agent's state to the new state. You'll need to take into consideration that your users could be on version `%0`, `%1` or have just started with the new version. `+on-load` will take the incoming state as a vase, switch on the head of the tagged union of the state (vase), and then update state to a `%2`-compliant version of the prior state's data, stored in the new state's data structure.

<table>
<tr>
<td>
:: initial version (lines 37-46)
</td>
<td>
:: new version (lines 35-46)
</td>
</tr>
<tr>
<td>

```hoon
  |=  incoming-state=vase
  ^-  (quip card _this)
  ~&  >  '%tudumvc has recompiled'
  =/  state-ver  !<(versioned-state incoming-state)
  ?-  -.state-ver
    %1
  `this(state state-ver)
    %0
  `this(state [%1 `tasks:tudumvc`(~(put by tasks) 1 [task.state-ver %.n])])
  ==
```
</td>
<td>

```hoon
  |=  incoming-state=vase
  ^-  (quip card _this)
  ~&  >  '%tudumvc has recompiled'
  =/  state-ver  !<(versioned-state incoming-state)
  ?-  -.state-ver
    %2
  `this(state state-ver)
    %1
  `this(state [%2 `shared-tasks:tudumvc`(~(put by shared) our.bowl tasks.state-ver) [~ ~ ~]])
    %0
  `this(state [%2 `shared-tasks:tudumvc`(~(put by shared) our.bowl (my :~([1 [task.state-ver %.n]]))) [~ ~ ~]])
  ==
```
</td>
</tr>
</table>

In the case where the saved state's head-tag is `%2`, keep the existing state.

In the case where the saved state's head-tag is `%1`, take the existing task list (which was individual; only for the host ship) and add it to `shared` in the state (which is a `(map ship tasks)`) with the host's ship name (`our.bowl`) as the key and the existing task list as the value. Also bunt (provide an empty tuple of sets to) the `editors` data structure, using with `[~ ~ ~]`.

In the case where the saved state's head-tag is `%0`, first make the single-item task into a `(map id [label done])` using `(my :~([1 [task.state-ver %n]]))`, then perform the `%1` manipulations to that map.

#### `++  on-poke` {#lesson-pt-I-pokes}
You need to make a few of changes to `+on-poke` to accommodate the changes made above:
* `action:tudumvc` poke additions:
  * `[%sub partner=ship]`
  * `[%unsub partner=ship]`
  * `[%force-remove paths=(list path) partner=ship]`
  * `[edit partners=(list ship) status=?(%approve %deny)]`

Also, as this section is getting a little long while still being fairly high up in the program, you'll want to offboard the working elements to `hc`, the helper core. This helps with legibility and doesn't impact the functionality. You can do this by making an arm in the helper core to handle each poke action and then call it using the `(arm data data1 data2)` form of [`%-`](https://urbit.org/docs/hoon/reference/rune/cen/#cenhep).

Your `+on-poke` arm changes will look like this:

<table>
<tr>
<td>
:: initial version (lines 48-101)
</td>
<td>
:: new version (lines 48-85)
</td>
</tr>
<tr>
<td>

```hoon
  |=  [=mark =vase]
  ^-  (quip card _this)
  |^
  =^  cards  state
  ?+  mark  (on-poke:def mark vase)
      %tudumvc-action  (poke-actions !<(action:tudumvc vase))
  ==
  [cards this]
  ::
  ++  poke-actions
    |=  =action:tudumvc
    ^-  (quip card _state)
    ?-  -.action
      %add-task
    =/  new-id=@ud
    ?~  tasks
      1
    +(-:(sort `(list @ud)`~(tap in ~(key by `(map id=@ud [task=@tU complete=?])`tasks)) gth))
    ~&  >  "Added task {<task.action>} at {<new-id>}"
    =.  state  state(tasks (~(put by tasks) new-id [+.action %.n]))
    :_  state
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc tasks)))]]]
    ::
      %remove-task
    ?:  =(id.action 0)
      =.  state  state(tasks ~)
      :_  state
      ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc tasks)))]]]
    ?.  (~(has by tasks) id.action)
      ~&  >>>  "No such task at ID {<id.action>}"
      `state
    =.  state  state(tasks (~(del by tasks) id.action))
    :_  state
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc tasks)))]]]
    ::
      %mark-complete
    ?.  (~(has by tasks) id.action)
      ~&  >>>  "No such task at ID {<id.action>}"
      `state
    =/  task-text=@tU  label.+<:(~(get by tasks) id.action)
    =/  done-state=?  ?!  done.+>:(~(get by tasks) id.action)
    ~&  >  "Task {<task-text>} marked {<done-state>}"
    =.  state  state(tasks (~(put by tasks) id.action [task-text done-state]))
    :_  state
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc tasks)))]]]
    ::
      %edit-task
    ~&  >  "Receiving facts {<id.action>} and {<label.action>}"
    =/  done-state=?  done.+>:(~(get by tasks) id.action)
    =.  state  state(tasks (~(put by tasks) id.action [label.action done-state]))
    :_  state
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc tasks)))]]]
    ==
  --
```
</td>
<td>

```hoon
  |=  [=mark =vase]
  ^-  (quip card _this)
  |^
  =^  cards  state
  ?+  mark  (on-poke:def mark vase)
      %tudumvc-action    (poke-actions !<(action:tudumvc vase))
  ==
  [cards this]
  ::
  ++  poke-actions
    |=  =action:tudumvc
    ^-  (quip card _state)
    =^  cards  state
    ?-  -.action
        %add-task
      ?>  |((team:title our.bowl src.bowl) (~(has in approved.editors) src.bowl))
      (add-task:hc our.bowl label.action)
        %remove-task
      ?>  |((team:title our.bowl src.bowl) (~(has in approved.editors) src.bowl))
      (remove-task:hc our.bowl id.action)
        %mark-complete
      ?>  |((team:title our.bowl src.bowl) (~(has in approved.editors) src.bowl))
      (mark-complete:hc our.bowl id.action)
        %edit-task
      ?>  |((team:title our.bowl src.bowl) (~(has in approved.editors) src.bowl))
      (edit-task:hc our.bowl id.action label.action)
        %sub
      (sub:hc partner.action)
        %unsub
      (unsub:hc partner.action)
        %force-remove
      ?>  (team:title our.bowl src.bowl)
      (force-remove:hc ~ partner.action)
        %edit
      ?>  (team:title our.bowl src.bowl)
      (edit:hc partners.action status.action)
    ==
    [cards state]
  --
```
</td>
</tr>
</table>

Making this section terse by offboarding to the helper core the changes made to state on each poke helps with legibility of the code.

The first change here is with the addition of the _assertions_ ([`?>`](https://urbit.org/docs/hoon/reference/rune/wut/#wutgar)) before some of the pokes. The assertion has a few parts that we should break down:
* [`?|`](https://urbit.org/docs/hoon/reference/rune/wut/#wutbar) (shorthand `|(condition condition1)`) allows us to check if any conditions provided to it are true and return true, if so. An OR statement, in other parlance.
* [`(team:title team-ship test-ship)`](https://github.com/urbit/urbit/blob/624e5c26929de6c666fa1585e2517f766bb8788b/pkg/arvo/sys/zuse.hoon#L4829) takes a "team" ship and a "test" ship (my terms, not documented in the code). It then checks to see if the "test" ship is in the "team" ship's "team". Basically, this just means "is this 'test' ship either the 'team' ship or one of it's subordinates (moon)?"
* `(~(has in approved.editors) src.bowl)` tests whether the source of the poke (src.bowl will always be the ship that sent the poke) is in the approved list of editors. If they are, this will return `%.y`, and if they are not, `%.n`.

The command `?> |((team:title our.bowl src.bowl) (~(has in approved.editors) src.bowl)` checks to make sure any pokes coming in for task actions (`%add` `%remove-task` `%mark-complete` `%edit-task`) come from either the host ship (or one of our subordinate moons), or an approved editor from the host's `editors` state data structure.
**NOTE:** The commands to enable editing or `%kick` (`%force-remove`) someone and to `%edit` someone's permissions in `editors` are reserved only for the "team" (the host ship or one of its subordinates).
**NOTE:** `%sub` and `%unsub` are wide open to any user utilizing them allow anyone to subscribe or unsubscribe to your task list (at least for now, imagine how you might modify the app to make subscriptions pendant on permissions as well).

The helper core should be updated to do all the work, as follows:

<table>
<tr>
<td>
:: add these arms to your helper core (starting on line 122 in the prior lesson's code)
</td>
</tr>
<td>

```hoon
++  add-task
  |=  [=ship label=@tu]
  ?.  (team:title our.bol ship)
    :_  state
    ~[[%pass /send-poke %agent [ship %tudumvc] %poke %tudumvc-action !>([%add-task label])]]
  =/  task-map=tasks:tudumvc  (~(got by shared) our.bol)
  =/  new-id=@ud
  ?~  task-map
    1
  +(-:`(list @ud)`(sort ~(tap in `(set id=@ud)`~(key by `tasks:tudumvc`task-map)) gth))
  ~&  >  "Added task {<label>} at id {<new-id>}"
  =.  state  state(shared (~(put by shared) our.bol (~(put by task-map) new-id [label %.n])))
  :_  state
  :~  [%give %fact ~[/sub-tasks] [%tudumvc-update !>((updates:tudumvc %task-add new-id label))]]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
++  remove-task
  |=  [=ship id=@ud]
  ?.  (team:title our.bol ship)
    :_  state
    ~[[%pass /send-poke %agent [ship %tudumvc] %poke %tudumvc-action !>([%remove-task id])]]
  ?:  =(id 0)
    =.  state  state(shared (~(put by shared) our.bol ~))
    ~&  >  "I'm here"
    :_  state
    :~  [%give %fact ~[/sub-tasks] [%tudumvc-update !>((updates:tudumvc %full-send ~))]]
        [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
    ==
  =/  task-map=tasks:tudumvc  (~(got by shared) our.bol)
  ?.  (~(has by task-map) id)
    ~&  >>>  "No task at id {<id>}"
    `state
  ~&  >  "Removing task at id {<id>} from your tasks"
  =.  state  state(shared (~(put by shared) our.bol (~(del by task-map) id)))
  :_  state
  :~  [%give %fact ~[/sub-tasks] [%tudumvc-update !>((updates:tudumvc %task-remove id))]]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
++  mark-complete
  |=  [=ship id=@ud]
  ?.  (team:title our.bol ship)
    :_  state
    ~[[%pass /send-poke %agent [ship %tudumvc] %poke %tudumvc-action !>([%mark-complete id])]]
  =/  task-map=tasks:tudumvc  (~(got by shared) our.bol)
  ?.  (~(has by task-map) id)
    ~&  >>>  "No task at id {<id>}"
    `state
  =/  task-text=@tU  label.+<:(~(get by task-map) id)
  =/  done-state=?  ?!  done.+>:(~(get by task-map) id)
  =.  state  state(shared (~(put by shared) our.bol (~(put by task-map) id [task-text done-state])))
  ~&  >  "Task {<task-text>} marked {<done-state>}"
  :_  state
  :~  [%give %fact ~[/sub-tasks] [%tudumvc-update !>((updates:tudumvc %task-complete id done-state))]]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
++  edit-task
  |=  [=ship id=@ud label=@tU]
  ?.  (team:title our.bol ship)
    :_  state
    ~[[%pass /send-poke %agent [ship %tudumvc] %poke %tudumvc-action !>([%edit-task id label])]]
  =/  task-map=tasks:tudumvc  (~(got by shared) our.bol)
  ?.  (~(has by task-map) id)
    ~&  >>>  "No such task at id {<id>}"
    `state
  ~&  >  "Task id {<id>} text changed to {<label>}"
  =/  done-state=?  done.+>:(~(get by task-map) id)
  =.  state  state(shared (~(put by shared) our.bol (~(put by task-map) id [label done-state])))
  :_  state
  :~  [%give %fact ~[/sub-tasks] [%tudumvc-update !>((updates:tudumvc %task-edit id label))]]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
++  sub
  |=  partner=ship
  ~&  >  "Subscribing to {<partner>}'s tasks"
  :_  state
  ~[[%pass /sub-tasks/(scot %p our.bol) %agent [partner %tudumvc] %watch /sub-tasks]]
++  unsub
  |=  partner=ship
  ~&  >  "Unsubscribing from {<partner>}'s tasks"
  =.  shared  (~(del by shared) partner)
  :_  state
  :~  [%pass /sub-tasks/(scot %p our.bol) %agent [partner %tudumvc] %leave ~]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
++  force-remove
  |=  [paths=(list path) partner=ship]
  =.  editors  [(~(dif in requested.editors) (sy ~[partner])) (~(dif in approved.editors) (sy ~[partner])) (~(dif in denied.editors) (sy ~[partner]))]
  :_  state
  ~[[%give %kick paths `partner]]
++  edit
  |=  [partners=(list ship) status=?(%approve %deny)]
  ?-  status
      %approve
    =.  editors  [(~(dif in requested.editors) (sy partners)) (~(gas in approved.editors) partners) (~(dif in denied.editors) (sy partners))]
    `state
      %deny
    =.  editors  [(~(dif in requested.editors) (sy partners)) (~(dif in approved.editors) (sy partners)) (~(gas in denied.editors) partners)]
    `state
  ==
```
</td>
</tr>
</table>

That may seem like a lot of code, but most of it you're already familiar with. If you compare the `+add-task` arm to our prior code, as below:

<table>
<tr>
<td>
:: initial version (lines 48-101)
</td>
<td>
:: new version (lines 48-85)
</td>
</tr>
<tr>
<td>

```hoon
      %add-task
    =/  new-id=@ud
    ?~  tasks
      1
    +(-:(sort `(list @ud)`~(tap in ~(key by `(map id=@ud [task=@tU complete=?])`tasks)) gth))
    ~&  >  "Added task {<task.action>} at {<new-id>}"
    =.  state  state(tasks (~(put by tasks) new-id [+.action %.n]))
    :_  state
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc tasks)))]]]
    ::
```
</td>
<td>

```hoon
++  add-task
  |=  [=ship label=@tu]
  ?.  (team:title our.bol ship)
    :_  state
    ~[[%pass /send-poke %agent [ship %tudumvc] %poke %tudumvc-action !>([%add-task label])]]
  =/  task-map=tasks:tudumvc  (~(got by shared) our.bol)
  =/  new-id=@ud
  ?~  task-map
    1
  +(-:`(list @ud)`(sort ~(tap in `(set id=@ud)`~(key by `tasks:tudumvc`task-map)) gth))
  ~&  >  "Added task {<label>} at id {<new-id>}"
  =.  state  state(shared (~(put by shared) our.bol (~(put by task-map) new-id [label %.n])))
  :_  state
  :~  [%give %fact ~[/sub-tasks] [%tudumvc-update !>((updates:tudumvc %task-add new-id label))]]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
```
</td>
</tr>
</table>

You should see that this isn't really that different. There are only two major changes:
1. Adding a test at the top to determine if the poke came from the host ship - but we're only passing `our.bowl` ever in `+on-poke`.
  * Later on, you'll add some Earth web functionality to use this test to `%give` a poke to some other ship's task list from the frontend - don't worry about this just yet.
2. %give`ing a `%fact` on a new path, `/sub-tasks` using the new mark (`updates`).
  * `%facts` passed on paths work exactly like what you were doing preivously to send your `tasks` list to your Earth web app, except you don't have to convert them to `%json`. 
  * Instead of `%json`, send a [cage](https://github.com/urbit/urbit/blob/624e5c26929de6c666fa1585e2517f766bb8788b/pkg/arvo/sys/arvo.hoon#L45) (or a pair of a mark and a vase) `%tudumvc-update` (which will use the update mark we just made) `!>((updates:tudumvc %task-add new-id label))` (which is a vase of `updates` tagged union type, specifically `%task-add`).
  * This `%fact` announces to subscribers when a task is added to the host's task list. This is mirrored in other changes to sends incremental updates as the host adds, removes, edits or marks complete its various tasks.

Incremental updates are great and save some computational power over sending all of the tasks as a package every time the host's state is updated. The incremental update will also send a task `id` for each incremental update just to make sure all ships stay in perfect pairity.

Nonetheless, the host should give its subscribers tge full task list at least once, when they first subscribe. The `hc` arm that handles `%sub` pokes will accommodate that:

```hoon
++  sub
  |=  partner=ship
  ~&  >  "Subscribing to {<partner>}'s tasks"
  :_  state
  ~[[%pass /sub-tasks/(scot %p our.bol) %agent [partner %tudumvc] %watch /sub-tasks]]
```

This part is a little confusing - you're poke-in your _own_ ship with `:tudumvc &tudumvc-action [%sub ~sampel-palnet]` to subscribe to (e.g.) `~sampel-palnet`'s task list. The arm handling that poke then tells your partner-ship that you want to subscribe using a card with the format `[%pass <wire> %agent [<partner-ship> <%app>] %watch <path>]`. This card will go to your partner-ship and tell them that you're going to start watching some path for data (note that `/sub-tasks` is the agent will send incremental updates on).

**NOTE:** The [`wire`](https://github.com/urbit/urbit/blob/624e5c26929de6c666fa1585e2517f766bb8788b/pkg/arvo/sys/arvo.hoon#L135) is also path, but that path must be unique to this individual subscription (at least until the subscription is terminated - some apps may terminate subscriptions or `%watch`es immediately, and so they can re-use the same wire repeatedly). You might think of it like this - the _path_ dictates _what data_ we'll receive, the _wire_ indicates _individual subscriptions_ on that path, on which the data will be received. Further, note that the wire could be completely different from the path (e.g., instead of `/sub-tasks/(scot %p our.bol)`,  something like `/arbitrary-wire/random-number`) - again, it's just a unique identifier for recipients of data sent along the path.

In the changes to `+on-watch`, a host receiving a request to `%watch` from some ship results in getting the full task list from the state of that ship, but it starts here with generating the `%watch`.

To finish the `hc` first, however, let's look at three further actions; `%unsub`, `%force-remove` and `%edit`.

```hoon
++  unsub
  |=  partner=ship
  ~&  >  "Unsubscribing from {<partner>}'s tasks"
  =.  shared  (~(del by shared) partner)
  :_  state
  :~  [%pass /sub-tasks/(scot %p our.bol) %agent [partner %tudumvc] %leave ~]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
```

`%unsub` has three important fuctions. When you unsubscribe from one of your partners, you'll first delete their task list from the `shared` data structure (`(~(del by shared) partner)`). Then you'll `[%pass <wire> %agent [<partner> <%app>] %leave ~]` to communicate to that partner that you're no longer listening on that path (freeing the wire for use by someone/something else, theoretically). Lastly, you'll update your Earth web app with the change to your state (removing those tasks from the Earth web app).

```hoon
++  force-remove
  |=  [paths=(list path) partner=ship]
  =.  editors  [(~(dif in requested.editors) (sy ~[partner])) (~(dif in approved.editors) (sy ~[partner])) (~(dif in denied.editors) (sy ~[partner]))]
  :_  state
  ~[[%give %kick paths `partner]]
```

`%force-remove` `%give`s a `%kick` to any subscriber - removing them from listening to the host on the path `/sub-tasks` and is the same as a non-optional `%unsub` poke to the subscriber-ship. The pattern for `%kick`s is `[%give %kick (list path) (unit ship)]`. A note here, better covered in [~timluc-miptev's Gall Guide](https://github.com/timlucmiptev/gall-guide/blob/master/poke.md#fact-or-kick-nuances), is that an empty (`~`) list of paths will, in this context, unsubscribe the user from _all_ paths to which they subscribe. In our context, there's only one real subscription action the user can take, so it doesn't make a lot of difference for us, but it's good to know.

In addition to `%kick`ing the partner from their subscription, `%force-remove` also removes the target from any/all allowances or denials they may have in the `editors` list, using the [`(~(dif in set) set1)`](https://urbit.org/docs/hoon/reference/stdlib/2h/#dif-in) set-logic function which computes the difference between two sets (providing the diverse elements of the first set). In other words, it checks to see whether the partner we're kicking is in each of the subsets of the tuple in our data element called `editors` and produces the result of the existing sub-set with that ship removed. 

```hoon
++  edit
  |=  [partners=(list ship) status=?(%approve %deny)]
  ?-  status
      %approve
    =.  editors  [(~(dif in requested.editors) (sy partners)) (~(gas in approved.editors) partners) (~(dif in denied.editors) (sy partners))]
    `state
      %deny
    =.  editors  [(~(dif in requested.editors) (sy partners)) (~(dif in approved.editors) (sy partners)) (~(gas in denied.editors) partners)]
    `state
  ==
```

`%edit` moves subscribers from the `%requested` set in the `editors` tuple to one of the other access levels - either approved on denied editors. It uses same `+dif:in` arm to manipulate the sets as in `%force-remove`. In addition, members are added to a given set using the standard library function [`gas:in`](https://urbit.org/docs/hoon/reference/stdlib/2h/#gas-in).

#### `++  on-peek`
We should probably update on-peek at some time, right? Maybe later.

#### `++  on-watch` {#lesson-pt-I-on-watch}
`+on-watch` is great, because it uses %galls built-in distributed-app functionality to handle data delivery. It listens for a path (as a gate that takes a path) in a `%watch` card. From the path, it knows what data to start delivering. In the last part of this guide, you implemented a path on which apps can listen, `/mytasks`, and receive `%json` data. In this version, you'll add `/sub-tasks` that returns task list data in a hoon format.

<table>
<tr>
<td>
:: initial version (lines 109-115)
</td>
<td>
:: new version (lines 114-125)
</td>
</tr>
<tr>
<td>

```hoon
  |=  =path
  ^-  (quip card _this)
  ?+  path  (on-watch:def path)
    [%mytasks ~]
  :_  this
  ~[[%give %fact ~[path] [%json !>((json (tasks-json:hc tasks)))]]]
  ==
```
</td>

<td>

```hoon
  |=  =path
  ^-  (quip card _this)
  =/  my-tasks=tasks:tudumvc  (~(got by shared) our.bowl)
  ?+  -.path  (on-watch:def path)
      %mytasks
    :_  this
    ~[[%give %fact ~ [%json !>((json (tasks-json:hc shared)))]]]
      %sub-tasks
    =.  requested.editors  (~(gas in requested.editors) ~[src.bowl])
    :_  this
    ~[[%give %fact ~ [%tudumvc-update !>((updates:tudumvc %full-send my-tasks))]]]
  ==
```
</td>
</tr>
</table>

The agent will send a card the first time a ship starts to `%watch` `/sub-tasks` using `[%give %fact ~ [%tudumvc-update !>((updates:tudumvc %full-send my-tasks))]]` - you'll see how that's handled by the subscriber in `+on-agent`, below.

The agent also, when hearing a `%watch` on `/sub-tasks`, adds the subscriber to the `requested.editors` state (letting you know they've asked for access to edit your tasks).

#### `++  on-agent` {#lesson-pt-I-on-agent}
In `+on-agent`, the agent handles cards coming from other ships on various wires. That is, subscriptions we've made will be sent data, and that data (sent in the form of `%fact`s or `%kick`s, in this case) will be processed using the code in this arm. `+on-agent` will need to handle:

* Our `updates:tudumvc` poke structure entirely:
  * `[%task-add id=@ud label=@tu]`
  * `[%task-remove id=@ud]`
  * `[%task-complete id=@ud done=?]`
  * `[%task-edit id=@ud label=@tU]`
  * `[%full-send =tasks]`
* Any `%kick`s we receive from ships we're subscribed to.

<table>
<tr>
<td>
:: initial version (line 118)
</td>
<td>
:: new version (lines 127-185)
</td>
</tr>
<tr>
<td>

```hoon
on-arvo:def
```
</td>

<td>

```hoon
  |=  [=wire =sign:agent:gall]
  ^-  (quip card _this)
  |^
  ?+  -.sign  (on-agent:def wire sign)
      %fact
    =/  update-action=updates:tudumvc  !<(updates:tudumvc q.cage.sign)
    =^  cards  this
    ?-  -.update-action
        %full-send
      ~&  >  "Receiving {<src.bowl>}'s task list"
      =.  shared  (~(put by shared) src.bowl tasks.update-action)
      :_  this
      ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc shared)))]]]
        %task-add
      ~&  >  "Receiving task {<label.update-action>} from {<src.bowl>}'s list"
      (partner-add id.update-action label.update-action src.bowl)
        %task-remove
      ~&  >  "Removing task {<id.update-action>} from {<src.bowl>}'s list"
      (partner-remove id.update-action src.bowl)
        %task-complete
      ~&  >  "Marking {<src.bowl>}'s task {<id.update-action>} as done: {<done.update-action>}"
      (partner-complete id.update-action done.update-action src.bowl)
        %task-edit
      ~&  >  "Editing {<src.bowl>}'s task {<id.update-action>} to read {<label.update-action>}"
      (partner-edit id.update-action label.update-action src.bowl)
    ==
    [cards this]
      %kick
    ~&  >  "{<src.bowl>} gave %kick - removing shared list"
    =.  shared  (~(del by shared) src.bowl)
    `this
  ==
  ++  partner-add
    |=  [id=@ud task=@tU =ship]
    =/  partner-list=tasks:tudumvc  (~(got by shared) ship)
    =.  shared  (~(put by shared) ship (~(put by partner-list) id [task %.n]))
    :_  this
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc shared)))]]]
  ++  partner-remove
    |=  [id=@ud =ship]
    =/  partner-list=tasks:tudumvc  (~(got by shared) ship)
    =.  shared  (~(put by shared) ship (~(del by partner-list) id))
    :_  this
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc shared)))]]]
  ++  partner-complete
    |=  [id=@ud done=? =ship]
    =/  partner-list=tasks:tudumvc  (~(got by shared) ship)
    =/  task=@tU  label:(~(got by partner-list) id)
    =.  shared  (~(put by shared) ship (~(put by partner-list) id [task done]))
    :_  this
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc shared)))]]]
  ++  partner-edit
    |=  [id=@ud task=@tU =ship]
    =/  partner-list=tasks:tudumvc  (~(got by shared) ship)
    =/  done=?  done:(~(got by partner-list) id)
    =.  shared  (~(put by shared) ship (~(put by partner-list) id [task done]))
    :_  this
    ~[[%give %fact ~[/mytasks] [%json !>((json (tasks-json:hc shared)))]]]
  --
```
</td>
</tr>
</table>

If the agent is receiving a `%fact` on the `/sub-tasks/...` wire that it created for a subscription, it should be one of the `updates:tudumvc` types. Most of these are handled in their own arm of the `+on-agent` `|^` core, but `%full-send` is handled in-line. 

`%full-send` takes in a full task list from some ship (`src.bowl`) and adds it to the `shared` data object in state by adding a key (of the ship) value (of the full task list) pair to that map.

All of the other incremental data types (`%task-add`, `%task-remove`, `%task-complete` and `%task-edit`) functional generally similar to how they do when poking your own ship for state changes. They even send the data to the Earth app on `/mytasks`.

If the agent receives a `%kick`, it cleans up the local state with [`del:by`](https://urbit.org/docs/hoon/reference/stdlib/2i/#del-by) and deletes the task list it was maintaining for that subscription.

#### ++  `on-leave` {#lesson-pt-I-on-leave}
`+on-leave` needs to handle requests being made to the agent to unsubscribe. On receiving unsubscribe actions, the agent will [`~&`](https://urbit.org/docs/hoon/reference/rune/sig/#sigpam) a message to the dojo indicating that some subscriber has left (utterly unnecessary but neat), and then remove the requesting ship from the `editors` data object entirely (so they aren't left with approved/denied edit rights, or left in the requested set).

**NOTE:** `+on-leave` takes a path. If your app had multiple subscription paths, you could use this to switch behavior on unsubscribe based on what data set the user is unsubscribing from.

<table>
<tr>
<td>
:: initial version (line 117)
</td>
<td>
:: new version (lines 187-191)
</td>
</tr>
<tr>
<td>

```hoon
on-leave:def
```
</td>

<td>

```hoon
  |=  =path
  ^-  (quip card _this)
  ~&  >>  "{<src.bowl>} has left the chat"
  =.  editors  [(~(dif in requested.editors) (sy ~[src.bowl])) (~(dif in approved.editors) (sy ~[src.bowl])) (~(dif in denied.editors) (sy ~[src.bowl]))]
  `this
```
</td>
</tr>
</table>

#### `++  tasks-json`
You also need to update `+tasks-json` to work with the new state data. Then you can update the Earth app to accommodate this data and to allow it to send task actions to remote ships (sort of - the pattern will actually be telling the host agent to tell the other ships to change the task).

The changes to `+tasks-json` are as follows:

<table>
<tr>
<td>
:: initial version (lines 123-139)
</td>
<td>
:: new version (lines 198-223)
</td>
</tr>
<tr>
<td>

```hoon
  |=  stat=tasks:tudumvc
  |^
  ^-  json
  =/  tasklist=(list [id=@ud label=@tU done=?])  ~(tap by stat)
  =/  objs=(list json)  (roll tasklist object-maker)
  [%a objs]
  ++  object-maker
  |=  [in=[id=@ud label=@tU done=?] out=(list json)]
  ^-  (list json)
  :-
  %-  pairs:enjs:format
    :~  ['done' [%b done.in]]
        ['id' [%s (scot %ud id.in)]]
        ['label' [%s label.in]]
    ==
  out
  --
```
</td>

<td>

```hoon
++  tasks-json
  |=  stat=shared-tasks:tudumvc
  |^
  ^-  json
  =/  shared-tasks=(list [partner=ship tasks=tasks:tudumvc])  ~(tap by stat)
  =/  objs=(list json)  (roll shared-tasks partner-handler)
  [%a objs]
  ++  partner-handler
    |=  [in=[partner=ship tasks=tasks:tudumvc] out=(list json)]
    ^-  (list json)
    =/  partners-tasks=(list [id=@ud label=@tU done=?])  ~(tap by tasks.in)
    =/  objs=(list json)  (roll partners-tasks object-maker)
    :-
    %-  pairs:enjs:format
      :~  [`@tU`(scot %p partner.in) [%a objs]]  ==
    out
  ++  object-maker
    |=  [in=[id=@ud label=@tU done=?] out=(list json)]
    ^-  (list json)
    :-
    %-  pairs:enjs:format
      :~  ['done' [%b done.in]]
          ['id' [%s (scot %ud id.in)]]
          ['label' [%s label.in]]
      ==
    out
  --
```
</td>
</tr>
</table>

The big change here is to handle a `shared-tasks` input, or a `(map ship tasks)`, rather than just a `tasks` input, using `+partner-handler` (which just calls `+object-maker` recursively for each partner). `+object maker` works just like it did before, but it's called for each ship key in `shared`. This pattern creates an array of JSON objects structured like this: `{'~sampel-palnet': [{'id': 1, 'label': 'the label', 'done': FALSE} {'id': 2, 'label': 'the label 2', 'done': FALSE}]}`. On the Earth web side, you'll be able to switch between the elements of this array to show all of the task lists for the host ship and any ship to which it subscribes.

### Testing General Networking {#lesson-pt-I-urbit-networked-applications}
You now have all you need to establish basic `%gall`-side networking of the agent. While the frontend wont work just yet, you can test the `%gall` side - try a few of the following actions between your two test ships after syncing and `|commit`ing:

* Check your state:
`:tudumvc +dbug %state` should yield something like:
```hoon
>   [ %2
    shared
  { [ p=owner=~nus
        q
        task-list
      {[p=id=1 q=[label='example task' done=%.n]]}
    ]
  }
  editors=[requested={} approved={} denied={}]
]
```
We can see the new state here - great!

* Subscribe to a friend:
`:tudumvc &tudumvc-action [%sub ~zod]` will show, on the subscribing ship:
```hoon
>   "Subscribing to ~zod's tasks"
> :tudumvc &tudumvc-action [%sub ~zod]
>=
>   "Receiving ~zod's task list"
```
Checking the state again:
```hoon
>   [ %2
    shared
  { [ p=owner=~nus
        q
        task-list
      {[p=id=1 q=[label='example task' done=%.n]]}
    ]
    [ p=owner=~zod
        q
        task-list
      {[p=id=1 q=[label='example task' done=%.n]]}
    ]
  }
  editors=[requested={} approved={} denied={}]
]
```
The agent is now storing `~zod`'s list of tasks, alongside the host's (`~nus`).

And, if you were to check `~zod`'s state with `:tudumvc +dbug %state`:
```hoon
>   [ %2
    shared
  { [ p=owner=~zod
        q
        task-list
      {[p=id=1 q=[label='example task' done=%.n]]}
    ]
  }
  editors=[requested={~nus} approved={} denied={}]
]
```
`~zod`` has dutifully recorded `~nus` as a `requested.editor`.

* Approve an editor:
Running `:tudumvc &tudumvc-action [%edit ~[~nus] %approve]` on `~zod` will add `~nus` to the set of `approved.editors`. You can check `~zod`'s state after running this to confirm:
```hoon
>   [ %2
    shared
  { [ p=owner=~zod
        q
        task-list
      {[p=id=1 q=[label='example task' done=%.n]]}
    ]
  }
  editors=[requested={} approved={~nus} denied={}]
```

* Add a task as an approved editor:
From `~nus`, run `:~zod/tudumvc &tudumvc-action [%add-task 'test from ~nus']` and you'll see, first on `~zod`:
```
>=hoon
>   "Added task ~~test.from.~~nus at id 2"
```
And then on `~nus`:
```hoon
>=
>   "Receiving task 'test from ~nus' from ~zod's list"
```
Note the order of operations:
1. `~nus` sent the poke to `~zod` telling it to add a task
2. `~zod` checked to determine if `~nus` was in the approved list (line 63)
3. `~zod` performed the add action using the helper core
4. `~zod` passed the updated data back along the `/sub-tasks` path to `~nus`
5. `~nus` updated its stored registry of `~zod`'s tasks

* Kick `~nus` from subscribing, using `~zod`'s interface:
From `~zod`, run `:tudumvc &tudumvc-action [%force-remove ~ ~nus]`. `~nus` will produce:
```hoon
>   "~zod gave %kick - removing shared list"
```
And, if you check `~nus`'s state thereafter:
```hoon
>   [ %2
    shared
  { [ p=owner=~nus
        q
        task-list
      { [ p=id=1
            q
          [ label='example task'
            done=%.n
          ]
        ]
      }
    ]
  }
    editors
  [requested={} approved={} denied={}]
]
```

Fiddle around with this a little more and get comfortable with the way the ships are now interacting, and then continue on to Part II of the Lesson.

## The Lesson (Part II) {#lesson-pt-II}
In this, the last section of this guide, you're going to implement Earth web support for the changes we made in the last section. Let's start with the hoon changes:

### Adding Support for `frontend` Actions
We're going to use new pokes from the Earth web app to handle actions for our task lists and those to which we subscribe. We're doing it this way because airlock doesn't handle 'remote' pokes the way we did from dojo in the last section (e.g. `:~zod/tudumvc &tudumvc-action [%add-task 'test task from ~nus']`). Instead, we'll poke our hosting ship from the frontend w/ data about to whom the change in task data is directed and the host ship will then send a card (a poke) to that remote ship, or apply it to its own task list, as appropriate.

#### Creating A `frontend` Type
Add a `frontend` type to your /sur file. `frontend` is like the original `action` type, but each tagged union element will _also_ include a ship name. This will be used to direct these pokes to the appropriate task-holder. Note that you'll still have to have permission to edit a remote ship's tasks for these to actually process.

<table>
<tr>
<td>
:: adding frontend type to our sur file (wherever)
</td>
</tr>
<td>

```hoon
:: Creates a structure for front end usage that will specify the ship to poke on 
:: each action, allowing our host ship to then send cards to each recipient ship
:: 
+$  frontend
  $%
    [%add-task =ship label=@tU]
    [%remove-task =ship id=@ud]
    [%mark-complete =ship id=@ud]
    [%edit-task =ship id=@ud label=@tU]
  ==
```
</td>
</tr>
</table>

#### Adding `/mar/tudumvc/frontend.hoon`
Also, you'll need a /mar mark that will handle these new `frontend` type. This mark will need to handle incoming JSON data from our Earth web app, so you'll need conversions to hoon types in the `+grab` arm:

<table>
<tr>
<td>
:: create a file called `frontend.hoon` in your /mar/tudumvc folder
</td>
</tr>
<td>

```hoon
:: Creates a structure for front end usage that will specify the ship to poke on 
:: each action, allowing our host ship to then send cards to each recipient ship
:: 
::
::  tudumvc-frontend mar file
::
/-  tudumvc
=,  dejs:format
|_  front=frontend:tudumvc
++  grad  %noun
++  grow
  |%
  ++  noun  front
  --
++  grab
  |%
  ++  noun  frontend:tudumvc
  ++  json
    |=  jon=^json
    %-  frontend:tudumvc
    =<
    (front-end jon)
    |%
    ++  front-end
      %-  of
      :~  [%add-task (ot :~(['ship' (se %p)] ['label' so]))]
          [%remove-task (ot :~(['ship' (se %p)] ['id' ni]))]
          [%mark-complete (ot :~(['ship' (se %p)] ['id' ni]))]
          [%edit-task (ot :~(['ship' (se %p)] ['id' ni] ['label' so]))]
      ==
    --
  --
--
```
</td>
</tr>
</table>

By now, you should be able to fairly easily read the JSON parsing we're doing here. Even if you're struggling with it, you should be able to see that these produce data in noun format that is of the `frontend` type you just created.

### Updating `/app/tudumvc.hoon`
Updating the app will consist of updating `+on-poke` for `frontend` pokes.

#### ++  on-poke
First, change the expected marks the agent might see coming in to `+on-poke`:

<table>
<tr>
<td>
:: initial version (lines 52-54)
</td>
<td>
:: new version (lines 52-55)
</td>
</tr>
<tr>
<td>

```hoon
  ?+  mark  (on-poke:def mark vase)
      %tudumvc-action    (poke-actions !<(action:tudumvc vase))
  ==
```
</td>

<td>

```hoon
  ?+  mark  (on-poke:def mark vase)
      %tudumvc-action    (poke-actions !<(action:tudumvc vase))
      %tudumvc-frontend  (frontend-actions !<(frontend:tudumvc vase))
  ==
```
</td>
</tr>
</table>

Then, add an arm that will handle those `frontend` type pokes:

<table>
<tr>
<td>
:: add an arm under poke-actions called frontend-actions
</td>
</tr>
<td>

```hoon
  ++  frontend-actions
    |=  =frontend:tudumvc
    ^-  (quip card _state)
    =^  cards  state
    ?-  -.frontend
        %add-task
      ?>  (team:title our.bowl src.bowl)
      (add-task:hc ship.frontend label.frontend)
        %remove-task
      ?>  (team:title our.bowl src.bowl)
      (remove-task:hc ship.frontend id.frontend)
        %mark-complete
      ?>  (team:title our.bowl src.bowl)
      (mark-complete:hc ship.frontend id.frontend)
        %edit-task
      ?>  (team:title our.bowl src.bowl)
      (edit-task:hc ship.frontend id.frontend label.frontend)
    ==
    [cards state]
```
</td>
</tr>
</table>

Now the decision to add a ship argument to the helper-core arms that handle changes makes sense. Before, when only using dojo, you can simply poke a remote ship and it will know to apply the changes to itself. From the Earth web, however, you're poking _your own_ ship with data about the ship to which the change should apply. If you take a look at `+add-task:hc`, again:
```hoon
++  add-task
  |=  [=ship label=@tu]
  ?.  (team:title our.bol ship)
    :_  state
    ~[[%pass /send-poke %agent [ship %tudumvc] %poke %tudumvc-action !>([%add-task label])]]
  =/  task-map=tasks:tudumvc  (~(got by shared) our.bol)
  =/  new-id=@ud
  ?~  task-map
    1
  +(-:`(list @ud)`(sort ~(tap in `(set id=@ud)`~(key by `tasks:tudumvc`task-map)) gth))
  ~&  >  "Added task {<label>} at id {<new-id>}"
  =.  state  state(shared (~(put by shared) our.bol (~(put by task-map) new-id [label %.n])))
  :_  state
  :~  [%give %fact ~[/sub-tasks] [%tudumvc-update !>((updates:tudumvc %task-add new-id label))]]
      [%give %fact ~[/mytasks] [%json !>((json (tasks-json shared)))]]
  ==
```

_If_ the ship being passed in the poke is not the host, rather than updating the state directly, it creates a card to `%pass` a `%poke` to the destination ship using the original `action` type (e.g. `[%add task label]`).

Lastly, you should update the Earth web app.

### Updating TodoMVC JavaScript
This isn't a React.s tutorial, and I'm not entirely positive my implementation is the best methodology so we're going to focus in on the changes that work with airlock, rather than going through all of the changes made to React to support the incoming data. Grab the files from [here](https://github.com/rabsef-bicrym/tudumvc/tree/main/src-lesson6/todomvc-end).

#### Changes to Pokes

<table>
<tr>
<td>
:: initial version of addTodo
</td>
<td>
:: new version of addTodo
</td>
</tr>
<tr>
<td>

```
  const addTodo = (task) => {
    urb.poke({app: 'tudumvc', mark: 'tudumvc-action', json: {'add-task': task}})
  };
```
</td>

<td>

```
  const addTodo = (label) => {
    urb.poke({app: 'tudumvc', mark: 'tudumvc-frontend', json: {'add-task': {'ship': subList[selected][1], 'label': label}}})
  };
```
</td>
</tr>
</table>

The big change here is adding in a specified ship to the poke and changing the mark to `tudumvc-frontend`. The other poke functions need change in exactly the same way.

The data flow will be:
1. Earth web app allows you to select a ship.
  * You can also subscribe to a new ship from the Earth web app (see addSub function).
2. Earth web app displays the selected ship's task list.
3. User makes changes to the Earth web displayed tasks for some ship.
4. Earth web pokes host ship with:
  * The ship to whom the action should be sent.
  * The action data itself.
5. Host ship's app determines how to direct that action and either applies it locally or passes it as a poke to the recipient.
6. Granting that the host ship is an approved editor for the recipient, the change is applied.
7. Recipient ship sends back updates on the `/sub-tasks` path

And, just like that, you've created a networked version of TodoMVC using Urbit as our data hosting platform. TodoMVC is dead, long live `%tudumvc`.

![finished product image](./final-product.png)

## Homework {#homework}
No homework this time - maybe in our next course!

## Exercises {#exercises}
* Beautify the display of tuduMVC in browser by cleaning up my pitiable implementation of the drop-down menu and the input box.
    * See if you can add ship name validation to the subscription box
* Add to the data being sent to the site a list of people requesting edit access to your task list, and a method whereby you can provide them edit access or deny their request
* Add an approval list for people requesting access to your todo list so not just anyone can subscribe.
    * This will require several other changes, in addition to just having the list.

## Summary and Addenda {#summary}
Well - you did it. You've just built your first Earth web app supported entirely by Urbit, with networking. 

Should you have any questions, please reach out to `~rabsef-bicrym` via Urbit DM and he'll try to get back to you quickly.

We hope you had fun and learned at least a few things. Maybe you've even been inspired to build your next project on Urbit! If you do build something, please make sure to let us know!