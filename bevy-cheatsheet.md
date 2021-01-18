# Bevy Cheatsheet

Concise cheat sheet to show the exact syntax for common features and programming patterns in the [Bevy game engine](https://github.com/bevyengine/bevy).

Primarily covers the current bevy release (v0.4), but new changes in the latest version (git) are also noted where applicable.

This document is maintained on a best-effort basis. Some information may be out of date.

Please help improve it and keep it up to date by contributing on [GitHub](https://github.com/jamadazi/bevy-cheatsheet).

If you like this, you should also have a look at the [Bevy Cookbook](https://github.com/jamadazi/bevy-cookbook).

## Entities

Conceptually, an "object" in the game.

Technically, just an ID to access component data -- type: `Entity`.

## Components

Components are per-entity data.

Defined as simple Rust structs. Accessed using queries.

```rust
struct MyComponent {
    stuff: u32,
    val: f32,
}
```

Empty (zero-sized) structs can be used as "marker components" to identify/tag entities (useful in `With`/`Without` queries, see below).

```rust
struct EmptyMarker;
```

Components are identified by their type; an entity cannot have more than one instance of the same component type. If you need more, you can wrap them in newtype structs, to create unique types.

## Component Bundles

Components can be grouped into bundles to make it easy to spawn an entity with a common set of components.

```rust
#[derive(Bundle)]
struct MyComponentsABC {
    a: ComponentA,
    b: ComponentB,
    c: ComponentC,
}
```

## Resources

Akin to "global variables", used to hold data independent of entities.

Defined as simple Rust structs. Accessed using `Res`/`ResMut` parameters.

Resources are identified by their type; there cannot be more than one resource of the same type. If you need more, you can wrap them in newtype structs, to create unique types.

### Resource Initialization

You can initialize your resources with `FromResources`:

```rust
struct MyFancyResource { /* stuff */ }

impl FromResources for MyFancyResource {
    fn from_resources(resources: &Resources) -> Self {
        // you have full access to any other resources you need here,
        // you can even mutate them:
        let mut x = resources.get_mut::<MyOtherResource>().unwrap();
        x.do_mut_stuff();

        MyFancyResource { /* stuff */ }
    }
}
```

If you don't need access to other resources, you can instead implement or derive `Default`:

```rust
#[derive(Default)]
struct SimpleResource {
    float: f32,
    opt: Option<f32>,
    v: Vec<f32>,
}
```

You can also construct your value and add it via the `App` builder.

You can also insert resources from a startup system, using `Commands`.

## Systems

Systems are regular rust functions that take special types as arguments.

Used to operate on game state by accessing components and resources.

They need to be registered at `App` creation time. 

## Queries

Queries allow you to operate on entities with a given set of components.

You can get mutable/immutable access to specific components.

You can also use `Option` as an adapter to get a component if it exists.

```rust
fn my_complex_system(
    q1: Query<(&mut ComponentA, &ComponentB)>,

    // use `Entity` to get the entity ID
    q2: Query<(Entity, &mut ComponentC)>,

    // optionally get mut access to `Cde` if it exists on entities that have `Bcd`
    q5: Query<(&Bcd, Option<&mut Cde>)>,
) {
    // iterate over all matching entities
    for (mut a, b) in q1.iter_mut() {
        // `a` is a special wrapper type (`Mut<T>`) that derefs to the actual component
        *a = ComponentA::new();
    }

    // or query just one specific entity
    if let Ok((mut a, b)) = q1.get(my_entity) {
        // do stuff with the components
    }

    // or just one component of a specific entity
    if let Ok(mut c) = q2.get_component_mut::<ComponentC>(my_entity) {
        // do stuff with e's ComponentC
    }
}
```

## Query Filters

Query types can optionally include filters, to apply additional restrictions to select what entities should match.

The available filters are: `With`, `Without`, `Mutated`, `Added`, `Changed`.

Multiple filters can be combined with a tuple to apply all of them (AND logic),
but also using the `Or<...>` wrapper to match on any of them (OR logic).

Examples:

```rust
// mutable access to Foo, but only if it has changed
Query<&mut Foo, Changed<Foo>>

// access Bar and Baz, but only if the entity does not have a Foo and its Qux was mutated
Query<(&Bar, &Baz), (Without<Foo>, Mutated<Qux>)>

// mutably access Abc if the entity has either Def or Fed
Query<&mut Abc, Or<With<Def>, With<Fed>>>

// regular query with no filter (the filter arg is optional)
Query<(&DataA, &mut DataB, &DataC)>
```

## Resource Change detection

Resources have basic change detection using `ChangedRes<T>`:

```rust
// the whole system only runs if the resource was changed
fn res_changed_system(my_res: ChangedRes<MyRes>) {
    eprintln!("Resource changed to {:?}", *my_res);
}
```

## Conflicting queries

For safety reasons, a system cannot have multiple queries with mutability conflicts on the same components:

```rust
// disallowed (will fail to compile): because both queries want `&mut ComponentA`
fn my_system(mut q1: Query<(&mut ComponentA, &ComponentB)>, mut q2: Query<(&mut ComponentA, &ComponentB)>)
```

The solution is to wrap them in a `QuerySet`:

```rust
fn my_system(mut qs: QuerySet<(Query<(&mut ComponentA, &ComponentB)>, Query<(&mut ComponentA, &ComponentB)>)>) {
    for a in qs.q0_mut().iter_mut() {
    }

    for (a, b) in qs.q1_mut().iter_mut() {
    }
}
```

This ensures that only one of the conflicting queries can be used at the same time.

## Commands

Spawn and despawn entities, add/remove components, insert resources, using `Commands`.

```rust
fn manager_system(mut cmd: &mut Commands, data: Res<MyRes>, mut q: Query<(Entity, &Stuff)>) {
    // spawn an entity; takes a component bundle
    cmd.spawn(MyComponentsAbc {
        a: ComponentA::new(),
        b: ComponentB::new(),
        c: ComponentC::new()
    }).with(ExtraComponent::default()); // add an ExtraComponent

    // tuples are also component bundles
    cmd.spawn((Foo::default(), Bar::default(), data.stuff.clone()));

    // to get the entity id of the last spawned entity:
    let my_e = cmd.current_entity().unwrap();

    for (e, s) in &mut q.iter() {
        if s.is_bad() {
            cmd.despawn(e); // despawn the entity
        }
    }
}
```

*Note*: Systems that take `&mut Commands` cannot be run in parallel. Alternatively, you can use `Arc<Mutex<Commands>>`, but it has gotchas:
* You can cause a deadlock if you lock the mutex again in a system that already holds the lock.
* `Commands` methods that operate on the current entity, like `with` and `current_entity`, break if you release the lock in-between the call to `spawn` and calling them. (bevyengine/bevy#795)

## Local Resources

You can have per-system data using `Local<T>`.

```rust
fn my_system(data: Local<MyData>) { }
```

The resource type must implement `Default` or `FromResources`, as it needs to be automatically initialized.

If the same type is used in multiple systems, they will each get their own instance.

## Events

Send notifications between systems. Accessed using a `Events<T>` resource.

To receive, you need an `EventReader<T>`. It tracks the events processed. It's nice to have it as a `Local` resource.

Events don't persist. If receivers don't handle them every frame, they will be lost.

```rust
struct MyEvent;

fn my_recv_system(events: Res<Events<MyEvent>>, mut reader: Local<EventReader<MyEvent>>) {
    for ev in reader.iter(&events) {
        // do something with `ev`
    }
}

fn my_send_system(mut events: ResMut<Events<MyEvent>>) {
    events.send(MyEvent);
}
```

Event types must be registered when building the `App`.

## App Initialization (main function)

To enter the bevy runtime, you must build an `App`, register any plugins, events, resources, and systems, that you use, and call `run()`.

```rust
fn main() {
    App::build().add_plugins(DefaultPlugins)
        .add_event::<MyEvent>()
        // construct a value for the MyRes1 resource
        .add_resource::<MyRes1>(MyRes1::new())
        // if it implements `Default` or `FromResources`
        .init_resource::<MyRes2>()
        // runs once at startup, before normal systems
        .add_startup_system(my_setup_system.system())
        // runs each frame
        .add_system(my_game_system.system())
        .run();
}
```

## Plugins

As your app grows, it can be useful to make it more modular. You can split it into plugins:

```rust
struct MyPlugin;

impl Plugin for MyPlugin {
    fn build(&self, app: &mut AppBuilder) {
        // add things to your app here
        app
            .add_event::<MyEvent>()
            .init_resource::<MyRsrc>()
            .add_system(my_system.system());
    }
}

fn main() {
    App::build().add_plugins(DefaultPlugins)
        .add_plugin::<MyPlugin>();
}
```

## Assets

`Assets<T>` resources store the actual data. They are indexed using `Handle<T>`s, which are lightweight IDs for individual loaded assets. The `AssetServer` resource handles loading of assets into the engine.

```rust
struct MyComponent {
    asset: Handle<Texture>,
}

fn game_system(storage: Res<Assets<Texture>>, data: &MyComponent) {
    // assets are loaded in the background
    // it could be `None` if that hasn't completed yet
    if let Some(asset) = storage.get(&data.asset) {
        // do something with the asset data
    }
}

fn setup(commands: Commands, server: Res<AssetServer>) {
    let handle = server.load("texture.png");
    commands.spawn((MyComponent { asset: handle },));
}
```

You can also add values directly to the `Assets<T>` storage and bypass the `AssetServer`, if you have loaded/generated them in another way.

## Hierarchical Entities

Entities can be nested into parent/child hierarchies.

```rust
// spawn entity with a child
let e = commands
    .spawn(MyParentComponents::default())
    .with_children(|parent| {
        parent.spawn(MyChildComponents::default());
    });

// despawn entity together with its children
commands.despawn_recursive(entity);
```

The parent entity will have a `Children` component, which contains the entity ids of the children.

The children will have a `Parent` component, which holds the entity id of the parent.

### Hierarchical Transforms

To use transforms with hierarchical entities, you must add both a `GlobalTransform` and a `Transform` component.

The `GlobalTransform` will be managed by bevy internally.

The `Transform` is your local transform. For the children, it is relative to the parent.

## Useful built-in resources

 - `AssetServer`: use to load assets from disk
 - `Input<KeyCode>`, `Input<MouseButton>`: to check if keys/buttons are pressed
 - `Time`: frame delta time and overall running time
 - `Windows`: parameters of the open windows, such as dimensions

### Configuration resources

These built-in resources can be added when building your `App`, to configure various things:

 - `ClearColor`: set the clear color (background color)
 - `Msaa`: enable Multi-Sample Anti-Aliasing
 - `WindowDescriptor`: change parameters of the main window

Note that some of these must be added before the `DefaultPlugins` for their values to have effect.

## Useful built-in events

 - Input devices: `KeyboardInput`, `CursorMoved`, `MouseMotion`, `MouseButtonInput`, `MouseWheel`.

## Useful built-in component bundles

 - `Camera2dComponents`: orthographic 2d camera
 - `Camera3dComponents`: perspective 3d camera
 - `UiCameraComponents`: camera for displaying UI
 - `SpriteComponents`, `SpriteSheetComponents`: 2d sprites
 - `MeshComponents`: 3d meshes
 - `LightComponents`: 3d lights
 - `TextComponents`: UI text
 - `ButtonComponents`: UI button

## Useful built-in components

 - `Draw`: rendering state;  set visibility and enable transparency here
 - `Transform`: the transform (position, rotation, scale) of an object in the game world
 - `TextureAtlasSprite`: the sprite id in a spritesheet
 - `Timer`: detect when an interval of time has elapsed; can be repeating

## Useful built-in asset types

 - `Font`: font for rendering text
 - `Texture`: image data
 - `TextureAtlas`: spritesheet
 - `Mesh`: 3d geometry

## Useful built-in systems

Systems that come with bevy, but are not enabled by default. Add them to your App if you want them.

 - `bevy::input::system::exit_on_esc_system`: quit the app when pressing the Esc key.

## Syntax tricks

To work around limitations on the number of parameters, they can be arbitrarily nested in tuples:

```rust
fn many_resources(
    (res1, mut res11): (Res<Res1>, ResMut<Res11>),
    (res2, res12): (Res<Res2>, Res<Res12>),
    res3: Res<Res3>,
    mut res4: ResMut<Res4>,
    res5: Res<Res5>,
    res6: Res<Res6>,
    mut res7: ResMut<Res7>,
    res8: Res<Res8>,
    res9: Res<Res9>,
    res10: Res<Res10>, // the max number of resource parameters is 10
    // mut res11: ResMut<Res11>, // this would cause a compilation error
) { /* ... */ }
```

