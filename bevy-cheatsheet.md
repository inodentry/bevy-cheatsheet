# Bevy Cheatsheet

Concise cheat sheet to show the exact syntax for common features and programming patterns in the [Bevy game engine](https://github.com/bevyengine/bevy).

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

## Systems

Systems are regular rust functions that take special types as arguments.

Used to operate on game state by accessing components and resources.

They need to be registered at `App` creation time. 

*Note*: Bevy requires a specific ordering for the `fn` arguments: `Commands` first, followed by resources, followed by queries or components last.

## Queries

Queries allow you to operate on entities with a given set of components.

You can get mutable/immutable access to specific components.

You can filter based on presence/absence of a component type, using `With<T, ...>`/`Without<T, ...>`.

```rust
fn my_complex_system(
    // resources must come before queries
    r1: Res<MyRes>,
    mut r2: ResMut<MyOtherRes>,

    // queries must be defined as `mut`, even if the components themselves are not
    mut q1: Query<(&ComponentA, &ComponentB)>,
    // use `Entity` to get the entity ID
    mut q2: Query<(Entity, &mut ComponentC)>,
    // only get entities that have a `Foo` component
    mut q3: Query<With<Foo, (&Bar, &mut Baz)>>,
    // only get entities without a `Abc` component
    mut q3: Query<Without<Abc, (&mut Def, Entity)>>,
    // only entities whose `Qux` component was mutated by another system this frame
    mut q4: Query<(Mutated<Qux>, &OtherData)>,
) {
    // iterate over all matching entities
    for (mut a, b) in &mut q1.iter() {
    	// `a` and `b` are special wrapper types that deref to the actual components
	*a = ComponentA::new();

	let e = b.some_method_that_returns_entity_id();

	// operate on a component of a specific entity
	if let Ok(mut c) = q2.get_mut::<ComponentC>(e) {
	   // do stuff with e's ComponentC
	}
    }
}
```

## Commands

Spawn and despawn entities, add/remove components, insert resources, using `Commands`. If used, must be the first argument of the `fn`.

```rust
fn manager_system(mut cmd: Commands, data: Res<MyRes>, mut q: Query<(Entity, &Stuff)>) {
    // spawn an entity; takes a component bundle
    cmd.spawn(MyComponentsAbc {
        a: ComponentA::new(),
        b: ComponentB::new(),
        c: ComponentC::new()
    }).with(ExtraComponent::default()); // add an ExtraComponent

    // tuples are also component bundles
    cmd.spawn((Foo::default(), Bar::default(), data.stuff.clone()));

    for (e, s) in &mut q.iter() {
        if s.is_bad() {
            cmd.despawn(e); // despawn the entity
        }
    }
}
```

## For-each systems

Special kind of system to offer simpler syntax for iterating over a single query. Can still have resources and commands.

The query is handled internally by Bevy and the system is called for each entity with the given components.

You must use `Mut<T>` instead of `&mut T`.

```rust
fn my_system(a: &ComponentA, b: Mut<ComponentB>) { /* do stuff */ }
```

## Events

Send notifications between systems. Per frame, any events not handled are dropped by the next frame. Accessed using a `Events<T>` resource.

To read, you need an `EventReader<T>`. It tracks the events processed. It can be placed in your own resources, to have multiple, so different systems can each handle the same events.

Must be registered when constructing the `App`.

```rust
struct MyEvent;

struct MyResource {
    reader: EventReader<MyEvent>,
}

fn my_recv_system(mut state: ResMut<MyResource>, events: Res<Events<MyEvent>>) {
    for ev in state.reader.iter(&events) {
        // do something with `ev`
    }
}

fn my_send_system(mut events: ResMut<Events<MyEvent>>) {
    events.send(MyEvent);
}
```

## App Initialization (main function)

To enter the bevy runtime, you must build an `App`, register any plugins, events, resources, and systems, that you use, and call `run()`.

```rust
fn main() {
    App::build().add_default_plugins()
        .add_event::<MyEvent>()
        .add_resource::<MyRes1>(MyRes1::new())
        .init_resource::<MyRes2>() // if it implements `Default`
        // runs once at startup, before normal systems
        .add_startup_system(my_setup_system.system())
        // runs each frame
        .add_system(my_game_system.system())
	.run();
}
```

## Assets

`Assets<T>` resources store the actual data. They are indexed using `Handle<T>`s, which are lightweight IDs for individual loaded assets. The `AssetServer` resource handles loading of assets into the engine.

```rust
struct MyComponent {
    asset: Handle<MyData>,
}

fn game_system(storage: Res<Assets<MyData>>, data: &MyComponent) {
    // assets are loaded in the background
    // it could be `None` if that hasn't completed yet
    if let Some(asset) = storage.get(&data.asset) {
        // do something with the asset data
    }
}

fn setup(commands: Commands, server: Res<AssetServer>) {
    // replace unwrap with your own error handling
    let handle = server.load("path/to/asset").unwrap();
    commands.spawn((MyComponent { asset: handle },));
}
```

## Useful built-in resources

 - `AssetServer`: use to load assets from disk
 - `Input<KeyCode>`, `Input<MouseButton>`: to check if keys/buttons are pressed
 - `Time`: frame delta time and overall running time
 - `Windows`: parameters of the open windows, such as dimensions

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
 - `Translation`, `Rotation`, `Scale`: easy interface to the components of a transform
 - `Transform`: the actual transform matrix, autogenerated from the above components.
 - `TextureAtlasSprite`: the sprite id in a spritesheet
 - `Timer`: detect when an interval of time has elapsed; can be repeating

## Useful built-in asset types

 - `Font`: font for rendering text
 - `Texture`: image data
 - `TextureAtlas`: spritesheet
 - `Mesh`: 3d geometry

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

