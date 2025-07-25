---
title: Editor scripts
brief: This manual explains how to extend editor using Lua
---

# Editor scripts

You can create custom menu items and editor lifecycle hooks using Lua files with special extension: `.editor_script`. Using this system, you can tweak editor to enhance your development workflow.

## Editor script runtime

Editor scripts run inside an editor, in a Lua VM emulated by Java VM. All scripts share the same single environment, which means they can interact with each other. You can require Lua modules, just as with `.script` files, but Lua version that is running inside the editor is different, so make sure your shared code is compatible. Editor uses Lua version 5.2.x, more specifically [luaj](https://github.com/luaj/luaj) runtime, which is currently the only viable solution to run Lua on JVM. Besides that, there are some restrictions:
- there is no `debug` package;
- there is no `os.execute`, though we provide a similar `editor.execute()`;
- there is no `os.tmpname` and `io.tmpfile` — currently editor scripts can access files only inside the project directory;
- there is currently no `os.rename`, although we want to add it;
- there is no `os.exit` and `os.setlocale`.
- it's not allowed to use some long-running functions in contexts where the editor needs an immediate response from the script, see [Execution Modes](#execution-modes) for more details.

All editor extensions defined in editor scripts are loaded when you open a project. When you fetch libraries, extensions are reloaded, since there might be new editor scripts in a libraries you depend on. During this reload, no changes in your own editor scripts are picked up, since you might be in the middle of changing them. To reload them as well, you should run **Project → Reload Editor Scripts** command.

## Anatomy of `.editor_script`

Every editor script should return a module, like that:
```lua
local M = {}

function M.get_commands()
  -- TODO - define editor commands
end

function M.get_language_servers()
  -- TODO - define language servers
end

function M.get_prefs_schema()
  -- TODO - define preferences
end

return M
```
Editor then collects all editor scripts defined in project and libraries, loads them into single Lua VM and calls into them when needed (more on that in [commands](#commands) and [lifecycle hooks](#lifecycle-hooks) sections).

## Editor API

You can interact with the editor using `editor` package that defines this API:
- `editor.platform` — a string, either `"x86_64-win32"` for Windows, `"x86_64-macos"` for macOS or `"x86_64-linux"` for Linux.
- `editor.version` — a string, version name of Defold, e.g. `"1.4.8"`
- `editor.engine_sha1` — a string, SHA1 of Defold engine
- `editor.editor_sha1` — a string, SHA1 of Defold editor
- `editor.get(node_id, property)` — get a value of some node inside the editor. Nodes in the editor are various entities, such as script or collection files, game objects inside collections, json files loaded as resources, etc. `node_id` is a userdata that is passed to the editor script by the editor. Alternatively, you can pass resource path instead of node id, for example `"/main/game.script"`. `property` is a string. Currently these properties are supported:
  - `"path"` — file path from the project folder for *resources* — entities that exist as files or directories. Example of returned value: `"/main/game.script"`
  - `"children"` — list of children resource paths for directory resources
  - `"text"` — text content of a resource editable as text (such as script files or json). Example of returned value: `"function init(self)\nend"`. Please note that this is not the same as reading file with `io.open()`, because you can edit a file without saving it, and these edits are available only when accessing `"text"` property.
  - for atlases: `images` (list of editor nodes for images in the atlas) and `animations` (list of animation nodes)
  - for atlas animations: `images` (same as `images` in atlas)
  - for tilemaps: `layers` (list of editor nodes for layers in the tilemap)
  - for tilemap layers: `tiles` (an unbounded 2d grid of tiles), see `tilemap.tiles.*` for more info
  - for particlefx: `emitters` (list of emitter editor nodes) and `modifiers` (list of modifier editor nodes)
  - for particlefx emitters: `modifiers` (list of modifier editor nodes)
  - for collision objects: `shapes` (list of collision shape editor nodes)
  - for GUI files: `layers` (list of layer editor nodes)
  - some properties that are shown in the Properties view when you have selected something in the Outline view. These types of outline properties supported:
    - `strings`
    - `booleans`
    - `numbers`
    - `vec2`/`vec3`/`vec4`
    - `resources`
    - `curves`
    Please note that some of these properties might be read-only, and some might be unavailable in different contexts, so you should use `editor.can_get` before reading them and `editor.can_set` before making editor set them. Hover over property name in Properties view to see a tooltip with information about how this property is named in editor scripts. You can set resource properties to `nil` by supplying `""` value.
- `editor.can_get(node_id, property)` — check if you can get this property so `editor.get()` won't throw an error.
- `editor.can_set(node_id, property)` — check if `editor.tx.set()` transaction step with this property won't throw an error.
- `editor.create_directory(resource_path)` — create a directory if it does not exist, and all non-existent parent directories.
- `editor.delete_directory(resource_path)` — delete a directory if it exists, and all existent child directories and files.
- `editor.execute(cmd, [...args], [options])` — run a shell command, optionally capturing its output.
- `editor.save()` — persist all unsaved changed to disk.
- `editor.transact(txs)` — modify the editor in-memory state using 1 or more transaction steps created with `editor.tx.*` functions.
- `editor.ui.*` — various UI-related functions, see [UI manual](/manuals/editor-scripts-ui).
- `editor.prefs.*` — functions for interacting with editor preferences, see [preferences](#preferences).

You can find the full editor API reference [here](https://defold.com/ref/alpha/editor/).

## Commands

If editor script module defines function `get_commands`, it will be called on extension reload, and returned commands will be available for use inside the editor in menu bar or in context menus in Assets and Outline panes. Example:
```lua
local M = {}

function M.get_commands()
  return {
    {
      label = "Remove Comments",
      locations = {"Edit", "Assets"},
      query = {
        selection = {type = "resource", cardinality = "one"}
      },
      active = function(opts)
        local path = editor.get(opts.selection, "path")
        return ends_with(path, ".lua") or ends_with(path, ".script")
      end,
      run = function(opts)
        local text = editor.get(opts.selection, "text")
        editor.transact({
          editor.tx.set(opts.selection, "text", strip_comments(text))
        })
      end
    },
    {
      label = "Minify JSON",
      locations = {"Assets"},
      query = {
        selection = {type = "resource", cardinality = "one"}
      },
      active = function(opts)
        return ends_with(editor.get(opts.selection, "path"), ".json")
      end,
      run = function(opts)
        local path = editor.get(opts.selection, "path")
        editor.execute("./scripts/minify-json.sh", path:sub(2))
      end
    }
  }
end

return M
```
Editor expects `get_commands()` to return an array of tables, each describing a separate command. Command description consists of:

- `label` (required) — text on a menu item that will be displayed to the user
- `locations` (required) — an array of either `"Edit"`, `"View"`, `"Project"`, `"Debug"`, `"Assets"`, `"Bundle"`, `"Scene"` or `"Outline"`, describes a place where this command should be available. `"Edit"`, `"View"`, `"Project"` and `"Debug"` mean menu bar at the top, `"Assets"` means context menu in Assets pane, `"Outline"` means context menu in Outline pane, and `"Bundle"` means **Project → Bundle** submenu.
- `query` — a way for command to ask editor for relevant information and define what data it operates on. For every key in `query` table there will be corresponding key in `opts` table that `active` and `run` callbacks receive as argument. Supported keys:
  - `selection` means this command is valid when there is something selected, and it operates on this selection.
    - `type` is a type of selected nodes command is interested in, currently these types are allowed:
      - `"resource"` — in Assets and Outline, resource is selected item that has a corresponding file. In menu bar (Edit or View), resource is a currently open file;
      - `"outline"` — something that can be shown in the Outline. In Outline it's a selected item, in menu bar it's a currently open file;
      - `"scene"` — something that can be rendered to the Scene.
    - `cardinality` defines how many selected items there should be. If `"one"`, selection passed to command callback will be a single node id. If `"many"`, selection passed to command callback will be an array of one or more node ids.
  - `argument` — command argument. Currently, only commands in `"Bundle"` location receive an argument, which is `true` when the bundle command is selected explicitly and `false` on rebundle.
- `id` - command identifier string, used e.g. for persisting the last used bundle command in `prefs`
- `active` - a callback that is executed to check that command is active, expected to return boolean. If `locations` include `"Assets"`, `"Scene"` or `"Outline"`, `active` will be called when showing context menu. If locations include `"Edit"` or `"View"`, active will be called on every user interaction, such as typing on keyboard or clicking with mouse, so be sure that `active` is relatively fast.
- `run` - a callback that is executed when user selects the menu item.

### Use commands to change the in-memory editor state

Inside the `run` handler, you can query and change the in-memory editor state. Querying is done using `editor.get()` function, where you can ask the editor about the current state of files and selection (if using `query = {selection = ...}`). You can get the `"text"` property of script files, and also some properties shown in the Properties view — hover over property name to see a tooltip with information about how this property is named in editor scripts. Changing the editor state is done using `editor.transact()`, where you bundle 1 or more modifications in a single undoable step. For example, if you want to be able to reset transform of a game object, you could write a command like that:
```lua
{
  label = "Reset transform",
  locations = {"Outline"},
  query = {selection = {type = "outline", cardinality = "one"}},
  active = function(opts)
    local node = opts.selection
    return editor.can_set(node, "position") 
       and editor.can_set(node, "rotation") 
       and editor.can_set(node, "scale")
  end,
  run = function(opts)
    local node = opts.selection
    editor.transact({
      editor.tx.set(node, "position", {0, 0, 0}),
      editor.tx.set(node, "rotation", {0, 0, 0}),
      editor.tx.set(node, "scale", {1, 1, 1})
    })
  end
}
```

#### Editing atlases

In addition to reading and writing properties of an atlas, you can read and modify atlas images and animations. Atlas defines `images` and `animations` node list properties, and animations define `images` node list property: you can use `editor.tx.add`, `editor.tx.remove` and `editor.tx.clear` transaction steps with these properties.

For example, to add an image to an atlas, execute the following code in the command's `run` handler:
```lua
editor.transact({
    editor.tx.add("/main.atlas", "images", {image="/assets/hero.png"})
})
```
To find a set of all images in an atlas, execute the following code:
```lua
local all_images = {} ---@type table<string, true>
-- first, collect all "bare" images
local image_nodes = editor.get("/main.atlas", "images")
for i = 1, #image_nodes do
    all_images[editor.get(image_nodes[i], "image")] = true
end
-- second, collect all images used in animations
local animation_nodes = editor.get("/main.atlas", "animations")
for i = 1, #animation_nodes do
    local animation_image_nodes = editor.get(animation_nodes[i], "images")
    for j = 1, #animation_image_nodes do
        all_images[editor.get(animation_image_nodes[j], "image")] = true
    end
end
pprint(all_images)
-- {
--     ["/assets/hero.png"] = true,
--     ["/assets/enemy.png"] = true,
-- }}
```
To replace all animations in an atlas:
```lua
editor.transact({
    editor.tx.clear("/main.atlas", "animations"),
    editor.tx.add("/main.atlas", "animations", {
        id = "hero_run",
        images = {
            {image = "/assets/hero_run_1.png"},
            {image = "/assets/hero_run_2.png"},
            {image = "/assets/hero_run_3.png"},
            {image = "/assets/hero_run_4.png"}
        }
    })
})
```

#### Editing tilesources

In addition to outline properties, tilesources define the following properties:
- `animations` - a list of animation nodes of the tilesource
- `collision_groups` - a list of collision group nodes of the tilesource
- `tile_collision_groups` - a table of collision group assignments for tiles in the tilesource

For example, here is how you can setup a tilesource:
```lua
local tilesource = "/game/world.tilesource"
editor.transact({
    editor.tx.add(tilesource, "animations", {id = "idle", start_tile = 1, end_tile = 1}),
    editor.tx.add(tilesource, "animations", {id = "walk", start_tile = 2, end_tile = 6, fps = 10}),
    editor.tx.add(tilesource, "collision_groups", {id = "player"}),
    editor.tx.add(tilesource, "collision_groups", {id = "obstacle"}),
    editor.tx.set(tilesource, "tile_collision_groups", {
        [1] = "player",
        [7] = "obstacle",
        [8] = "obstacle"
    })
})
```

#### Editing tilemaps

Tilemaps define `layers` property, a node list of tilemap layers. Each layer also defines a `tiles` property that holds an unbounded 2d grid of tiles on this layer. This is different from the engine: tiles have no bounds and may be added anywhere, including negative coordinates. To edit tiles, the editor script API defines a `tilemap.tiles` module with the following functions:
- `tilemap.tiles.new()` to create a fresh data structure that holds an unbounded 2d tile grid (in the editor, contrary to the engine, the tilemap is unbounded, and coordinates may be negative)
- `tilemap.tiles.get_tile(tiles, x, y)` to get a tile index at a specific coordinate
- `tilemap.tiles.get_info(tiles, x, y)` to get full tile information at a specific coordinate (the data shape is the same as in the engine's `tilemap.get_tile_info` function)
- `tilemap.tiles.iterator(tiles)` to create an iterator over all tiles in the tilemap
- `tilemap.tiles.clear(tiles)` to remove all tiles from the tilemap
- `tilemap.tiles.set(tiles, x, y, tile_or_info)` to set a tile at a specific coordinate
- `tilemap.tiles.remove(tiles, x, y)` to remove a tile at a specific coordinate

For example, here is how you can print the contents of the whole tilemap:
```lua
local layers = editor.get("/level.tilemap", "layers")
for i = 1, #layers do
    local layer = layers[i]
    local id = editor.get(layer, "id")
    local tiles = editor.get(layer, "tiles")
    print("layer " .. id .. ": {")
    for x, y, tile in tilemap.tiles.iterator(tiles) do
        print("  [" .. x .. ", " .. y .. "] = " .. tile)
    end
    print("}")
end
```

Here is an example that shows how to add a layer with tiles to a tilemap:
```lua
local tiles = tilemap.tiles.new()
tilemap.tiles.set(tiles, 1, 1, 2)
editor.transact({
    editor.tx.add("/level.tilemap", "layers", {
        id = "new_layer",
        tiles = tiles
    })
})
```

#### Editing particlefx

You can edit particlefx using `modifiers` and `emitters` properties. For example, adding a circle emitter with acceleration modifier is done like this:
```lua
editor.transact({
    editor.tx.add("/fire.particlefx", "emitters", {
        type = "emitter-type-circle",
        modifiers = {
          {type = "modifier-type-acceleration"}
        }
    })
})
```
Many particlefx properties are curves or curve spreads (i.e. curve + some randomizer value). Curves are represented as a table with a non-empty list of `points`, where each point is a table with the following properties:
- `x` - the x coordinate of the point, should start at 0 and end at 1
- `y` - the value of the point
- `tx` (0 to 1) and `ty` (-1 to 1) - tangents of the point. E.g., for an 80-degree angle, `tx` should be `math.cos(math.rad(80))` and `ty` should be `math.sin(math.rad(80))`.
Curve spreads additionally have a `spread` number property. 

For example, setting a particle lifetime alpha curve for an already existing emitter might look like this:
```lua
local emitter = editor.get("/fire.particlefx", "emitters")[1]
editor.transact({
    editor.tx.set(emitter, "particle_key_alpha", { points = {
        {x = 0,   y = 0, tx = 0.1, ty = 1}, -- start at 0, go up quickly
        {x = 0.2, y = 1, tx = 1,   ty = 0}, -- reach 1 at 20% of a lifetime
        {x = 1,   y = 0, tx = 1,   ty = 0}  -- slowly go down to 0
    }})
})
```
Of course, it's also possible to use the `particle_key_alpha` key in a table when creating an emitter. Additionally, you can use a single number instead to represent a "static" curve.

#### Editing collision objects

In addition to default outline properties, collision objects define `shapes` node list property. Adding new collision shapes is done like this:
```lua
editor.transact({
    editor.tx.add("/hero.collisionobject", "shapes", {
        type = "shape-type-box" -- or "shape-type-sphere", "shape-type-capsule"
    })
})
```
Shape's `type` property is required during creation and cannot be changed after the shape is added. There are 3 shape types:
- `shape-type-box` - box shape with `dimensions` property
- `shape-type-sphere` - sphere shape with `diameter` property
- `shape-type-capsule` - capsule shape with `diameter` and `height` properties

#### Editing GUI files

In addition to outline properties, GUI nodes defines the following properties:
- `layers` — list of layer editor nodes (reorderable)
- `materials` — list of material editor nodes

It's possible to edit GUI layers using editor `layers` property, e.g.:
```lua
editor.transact({
    editor.tx.add("/main.gui", "layers", {name = "foreground"}),
    editor.tx.add("/main.gui", "layers", {name = "background"})
})
```
Additionally, it's possible to reorder layers:
```lua
local fg, bg = table.unpack(editor.get("/main.gui", "layers"))
editor.transact({
    editor.tx.reorder("/main.gui", "layers", {bg, fg})
})
```
Similarly, fonts, materials, textures, and particlefxs are edited using `fonts`, `materials`, `textures`, and `particlefxs` properties:
```lua
editor.transact({
    editor.tx.add("/main.gui", "fonts", {font = "/main.font"}),
    editor.tx.add("/main.gui", "materials", {name = "shine", material = "/shine.material"}),
    editor.tx.add("/main.gui", "particlefxs", {particlefx = "/confetti.material"}),
    editor.tx.add("/main.gui", "textures", {texture = "/ui.atlas"})
})
```
These properties don't support reordering.

Finally, you can edit GUI nodes using `nodes` list property, e.g.:
```lua
editor.transact({
    editor.tx.add("/main.gui", "nodes", {
        type = "gui-node-type-box",
        position = {20, 20, 20}
    }),
    editor.tx.add("/main.gui", "nodes", {
        type = "gui-node-type-template",
        template = "/button.gui"
    }),
})
```
Built-in node types are:
- `gui-node-type-box`
- `gui-node-type-particlefx`
- `gui-node-type-pie`
- `gui-node-type-template`
- `gui-node-type-text`

If you are using spine extension, you can also use `gui-node-type-spine` node type.

If the GUI file defines layouts, you can get and set the values from layouts using `layout:property` syntax, e.g.:
```lua
local node = editor.get("/main.gui", "nodes")[1]

-- GET:
local position = editor.get(node, "position")
pprint(position) -- {20, 20, 20}
local landscape_position = editor.get(node, "Landscape:position")
pprint(landscape_position) -- {20, 20, 20}

-- SET:
editor.transact({
    editor.tx.set(node, "Landscape:position", {30, 30, 30})
})
pprint(editor.get(node, "Landscape:position")) -- {30, 30, 30}
```

Layout properties that were set can be reset to their default values using `editor.tx.reset`:
```lua
print(editor.can_reset(node, "Landscape:position")) -- true
editor.transact({
    editor.tx.reset(node, "Landscape:position")
})
```
Template node trees can be read, but not edited — you can only set node properties of the template node tree:
```lua
local template = editor.get("/main.gui", "nodes")[2]
print(editor.can_add(template, "nodes")) -- false
local node_in_template = editor.get(template, "nodes")[1]
editor.transact({
    editor.tx.set(node_in_template, "text", "Button text")
})
print(editor.can_reset(node_in_template, "text")) -- true (overrides a value in the template)
```

#### Editing game objects

It's possible to edit components of a game object file using editor scripts. The components come in 2 flavors: referenced and embedded. Referenced components use type `component-reference` and act as references to other resources, only allowing overrides of go properties defined in scripts. Embedded components use types like `sprite`, `label`, etc., and allow editing of all properties defined in the component type, as well as adding sub-components like shapes of collision objects. For example, you can use the following code to set up a game object:
```lua
editor.transact({
    editor.tx.add("/npc.go", "components", {
        type = "sprite",
        id = "view"
    }),
    editor.tx.add("/npc.go", "components", {
        type = "collisionobject",
        id = "collision",
        shapes = {
            {
                type = "shape-type-box",
                dimensions = {32, 32, 32}
            }
        }
    }),
    editor.tx.add("/npc.go", "components", {
        type = "component-reference",
        path = "/npc.script"
        id = "controller",
        __hp = 100 -- set a go property defined in the script
    })
})
```

#### Editing collections
It's possible to edit collections using editor scripts. You can add game objects (embedded or referenced) and collections (referenced). For example:
```lua
local coll = "/char.collection"
editor.transact({
    editor.tx.add(coll, "children", {
        -- embbedded game object
        type = "go",
        id = "root",
        children = {
            {
                -- referenced game object
                type = "go-reference",
                path = "/char-view.go"
                id = "view"
            },
            {
                -- referenced collection
                type = "collection-reference",
                path = "/body-attachments.collection"
                id = "attachments"
            }
        },
        -- embedded gos can also have components
        components = {
            {
                type = "collisionobject",
                id = "collision",
                shapes = {
                    {type = "shape-type-box", dimensions = {2.5, 2.5, 2.5}}
                }
            },
            {
                type = "component-reference",
                id = "controller",
                path = "/char.script",
                __hp = 100 -- set a go property defined in the script
            }
        }
    })
})
```

Like in the editor, referenced collections can only be added to the root of the edited collection, and game objects can only be added to embedded or referenced game objects, but not to referenced collections or game objects within these referenced collections.

### Use shell commands

Inside the `run` handler, you can write to files (using `io` module) and execute shell commands (using `editor.execute()` command). When executing shell commands, it's possible to capture the output of a shell command as a string and then use it in code. For example, if you want to make a command for formatting JSON that shells out to globally installed [`jq`](https://jqlang.github.io/jq/), you can write the following command:
```lua
{
  label = "Format JSON",
  locations = {"Assets"},
  query = {selection = {type = "resource", cardinality = "one"}},
  action = function(opts)
    local path = editor.get(opts.selection, "path")
    return path:match(".json$") ~= nil
  end,
  run = function(opts)
    local text = editor.get(opts.selection, "text")
    local new_text = editor.execute("jq", "-n", "--argjson", "data", text, "$data", {
      reload_resources = false, -- don't reload resources since jq does not touch disk
      out = "capture" -- return text output instead of nothing
    })
    editor.transact({ editor.tx.set(opts.selection, "text", new_text) })
  end
}
```
Since this command invokes shell program in a read-only way (and notifies the editor about it using `reload_resources = false`), you get the benefit of making this action undoable.

::: sidenote
If you want to distribute your editor script as a library, you might want to bundle the binary program for editor platforms within the dependency. See [Editor scripts in libraries](#editor-scripts-in-libraries) for more details on how to do it.
:::

## Lifecycle hooks

There is a specially treated editor script file: `hooks.editor_script`, located in a root of your project, in the same directory as *game.project*. This and only this editor script will receive lifecycle events from the editor. Example of such file:
```lua
local M = {}

function M.on_build_started(opts)
  local file = io.open("assets/build.json", "w")
  file:write('{"build_time": "' .. os.date() .. '"}')
  file:close()
end

return M
```
We decided to limit lifecycle hooks to single editor script file because order in which build hooks happen is more important than how easy it is to add another build step. Commands are independent from each other, so it does not really matter in what order they are shown in the menu, in the end user executes a particular command they selected. If it was possible to specify build hooks in different editor scripts, it would create a problem: in which order do hooks execute? You probably want to create a checksums of content after you compress it... And having a single file that establishes order of build steps by calling each step function explicitly is a way to solve this problem.

Existing lifecycle hooks that `/hooks.editor_script` may specify:
- `on_build_started(opts)` — executed when game is Built to run locally or on some remote target using either the Project Build or Debug Start options. Your changes will appear in the built game. Raising an error from this hook will abort a build. `opts` is a table that contains following keys:
  - `platform` — a string in `%arch%-%os%` format describing what platform it's built for, currently always the same value as in `editor.platform`.
- `on_build_finished(opts)` — executed when build is finished, be it successful or failed. `opts` is a table with following keys:
  - `platform` — same as in `on_build_started`
  - `success` — whether build is successful, either `true` or `false`
- `on_bundle_started(opts)` — executed when you create a bundle or Build HTML5 version of a game. As with `on_build_started`, changes triggered by this hook will appear in a bundle, and errors will abort a bundle. `opts` will have these keys:
  - `output_directory` — a file path pointing to a directory with bundle output, for example `"/path/to/project/build/default/__htmlLaunchDir"`
  - `platform` — platform the game is bundled for. See a list of possible platform values in [Bob manual](/manuals/bob).
  - `variant` — bundle variant, either `"debug"`, `"release"` or `"headless"`
- `on_bundle_finished(opts)` — executed when bundle is finished, be it successful or not. `opts` is a table with the same data as `opts` in `on_bundle_started`, plus `success` key indicating whether build is successful.
- `on_target_launched(opts)` — executed when user launched a game and it successfully started. `opts` contains an `url` key pointing to a launched engine service, for example, `"http://127.0.0.1:35405"`
- `on_target_terminated(opts)` — executed when launched game is closed, has same opts as `on_target_launched`

Please note that lifecycle hooks currently are an editor-only feature, and they are not executed by Bob when bundling from command line.

## Language servers

The editor supports a subset [Language Server Protocol](https://microsoft.github.io/language-server-protocol/). While we aim to expand the editor's support for LSP features in the future, currently it can only show diagnostics (i.e. lints) in the edited files and provide completions.

To define the language server, you need to edit your editor script's `get_language_servers` function like so:

```lua
function M.get_language_servers()
  local command = 'build/plugins/my-ext/plugins/bin/' .. editor.platform .. '/lua-lsp'
  if editor.platform == 'x86_64-win32' then
    command = command .. '.exe'
  end
  return {
    {
      languages = {'lua'},
      watched_files = {
        { pattern = '**/.luacheckrc' }
      },
      command = {command, '--stdio'}
    }
  }
end
```
The editor will start the language server using the specified `command`, using the server process's standard input and output for communication.

Language server definition table may specify:
- `languages` (required) — a list of languages the server is interested in, as defined [here](https://code.visualstudio.com/docs/languages/identifiers#_known-language-identifiers) (file extensions also work);
- `command` (required) - an array of command and its arguments
- `watched_files` - an array of tables with `pattern` keys (a glob) that will trigger the server's [watched files changed](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_didChangeWatchedFiles) notification.

## HTTP server

Every running instance of the editor has an HTTP server running. The server can be extended using editor scripts. To extend the editor HTTP server, you need to add `get_http_server_routes` editor script function — it should return the additional routes:
```lua
print("My route: " .. http.server.url .. "/my-extension")

function M.get_http_server_routes()
  return {
    http.server.route("/my-extension", "GET", function(request)
      return http.server.response(200, "Hello world!")
    end)
  }
end
```
After reloading the editor scripts, you'll see the following output in the console: `My route: http://0.0.0.0:12345/my-extension`. If you open this link in the browser, you'll see your `"Hello world!"` message.

The input `request` argument is a simple Lua table with information about the request. It contains keys such as `path` (URL path segment that starts with `/`), request `method` (e.g. `"GET"`), `headers` (a table with lower-case header names), and optionally `query` (the query string) and `body` (if the route defines how to interpret the body). For example, if you want to make a route that accepts JSON body, you define it with a `"json"` converter parameter:
```lua
http.server.route("/my-extension/echo-request", "POST", "json", function(request)
  return http.server.json_response(request)
end)
```
You can test this endpoint in the command line using `curl` and `jq`:
```sh
curl 'http://0.0.0.0:12345/my-extension/echo-request?q=1' -X POST --data '{"input": "json"}' | jq
{
  "path": "/my-extension/echo-request",
  "method": "POST",
  "query": "q=1",
  "headers": {
    "host": "0.0.0.0:12345",
    "content-type": "application/x-www-form-urlencoded",
    "accept": "*/*",
    "user-agent": "curl/8.7.1",
    "content-length": "17"
  },
  "body": {
    "input": "json"
  }
}
```
The route path supports patterns that can be extracted from the request path and provided to the handler function as a part of the request, e.g.:
```lua
http.server.route("/my-extension/setting/{category}.{key}", function(request)
  return http.server.response(200, tostring(editor.get("/game.project", request.category .. "." .. request.key)))
end)
```
Now, if you open e.g. `http://0.0.0.0:12345/my-extension/setting/project.title`, you'll see the title of your game taken from the `/game.project` file.

In addition to a single segment paths pattern, you can also match the rest of the URL path using `{*name}` syntax. For example, here is a simple file server endpoint that serves files from the project root:
```lua
http.server.route("/my-extension/files/{*file}", function(request)
  local attrs = editor.external_file_attributes(request.file)
  if attrs.is_file then
    return http.server.external_file_response(request.file)
  else
    return 404
  end
end)
```
Now, opening e.g. `http://0.0.0.0:12345/my-extension/files/main/main.collection` in the browser will display the contents of the `main/main.collection` file.

## Editor scripts in libraries

You can publish libraries for other people to use that contain commands, and they will be automatically picked up by the editor. Hooks, on the other hand, can't be picked up automatically, since they have to be defined in a file that is in a root folder of a project, but libraries expose only subfolders. This is intended to give more control over build process: you still can create lifecycle hooks as simple functions in `.lua` files, so users of your library can require and use them in their `/hooks.editor_script`.

Also note that although dependencies are shown in Assets view, they do not exist as files (they are entries in a zip archive). It's possible to make the editor extract some files from the dependencies into `build/plugins/` folder. To do it, you need to create `ext.manifest` file in your library folder, and then create `plugins/bin/${platform}` folder in the same folder where the `ext.manifest` file is located. Files in that folder will be automatically extracted to `/build/plugins/${extension-path}/plugins/bin/${platform}` folder, so your editor scripts can reference them.

## Preferences

Editor scripts can define and use preferences — persistent, uncommitted pieces of data stored on the user's computer. These preferences have three key characteristics:
- typed: every preference has a schema definition that includes the data type and other metadata like default value
- scoped: preferences are scoped either per project or per user
- nested: every preference key is a dot-separated string, where the first path segment identifies an editor script, and the rest 

All preferences must be registered by defining their schema:
```lua
function M.get_prefs_schema()
  return {
    ["my_json_formatter.jq_path"] = editor.prefs.schema.string(),
    ["my_json_formatter.indent.size"] = editor.prefs.schema.integer({default = 2, scope = editor.prefs.SCOPE.PROJECT}),
    ["my_json_formatter.indent.type"] = editor.prefs.schema.enum({values = {"spaces", "tabs"}, scope = editor.prefs.SCOPE.PROJECT}),
  }
end
```
After such editor script is reloaded, the editor registers this schema. Then the editor script can get and set the preferences, e.g.:
```lua
-- Get a specific preference
editor.prefs.get("my_json_formatter.indent.type")
-- Returns: "spaces"

-- Get an entire preference group
editor.prefs.get("my_json_formatter")
-- Returns:
-- {
--   jq_path = "",
--   indent = {
--     size = 2,
--     type = "spaces"
--   }
-- }

-- Set multiple nested preferences at once
editor.prefs.set("my_json_formatter.indent", {
    type = "tabs",
    size = 1
})
```

## Execution modes

The editor script runtime uses 2 execution modes that are mostly transparent to editor scripts: **immediate** and **long-running**. 

**Immediate** mode is used when the editor needs to receive a response from the script as fast as possible. For instance, menu commands' `active` callbacks are executed in immediate mode, because these checks are performed on the editors UI thread in response to user interacting with the editor, and should update the UI within the same frame. 

**Long-running** mode is used when the editor doesn't need an instantaneous response from the script. For example, menu commands' `run` callbacks are executed in a **long-running** mode, allowing the script to take more time to complete its work.

Some of the functions that the editor scripts can use may take a lot of time to run. For example, `editor.execute("git", "status", {reload_resources=false, out="capture"})` can take up to a second on sufficiently large projects. To maintain editor responsiveness and performance, functions that may be time-consuming are not allowed in contexts where the editor needs an immediate response. Attempting to use such a function in an immediate context will result in an error: `Cannot use long-running editor function in immediate context`. To resolve this error, avoid using such functions in immediate contexts.

The following functions are considered long-running and cannot be used in immediate mode:
- `editor.create_directory()`, `editor.delete_directory()`, `editor.save()`, `os.remove()` and `file:write()`: these functions modify the files on disc, causing the editor to synchronize its in-memory resource tree with the disc state, which can take seconds in large projects.
- `editor.execute()`: execution of shell commands can take an unpredictable amount of time.
- `editor.transact()`: large transactions on widely-referenced nodes may take hundreds of milliseconds, which is too slow for UI responsiveness.

The following code execution contexts use immediate mode:
- Menu command's `active` callbacks: the editor needs a response from the script within the same UI frame.
- Top-level of editor scripts: we don't expect the act of reloading editor scripts to have any side effects.

## Actions

::: sidenote
Previously, the editor interacted with the Lua VM in a blocking way, so there was a hard requirement for editor scripts to not block, since some interactions have to be done from the editor UI thread. For that reason, there was e.g. no `editor.execute()` and `editor.transact()`. Executing scripts and changing the editor state was instead triggered by returning an array of "actions" from hooks and command `run` handlers.

Now the editor interacts with the Lua VM in a non-blocking way, so there is no need for these actions any more: using functions like `editor.execute()` is more convenient, concise, and powerful. The actions are now **DEPRECATED**, though we have no plans to remove them.
:::

Editor scripts may return an array of actions from a command's `run` function or from `/hooks.editor_script`'s hook functions. These actions will then be performed by the editor.

Action is a table describing what editor should do. Every action has an `action` key. Actions come in 2 flavors: undoable and non-undoable.

### Undoable actions

::: sidenote
Prefer using `editor.transact()`.
:::

Undoable action can be undone after it is executed. If a command returns multiple undoable actions, they are performed together, and get undone together. You should use undoable actions if you can. Their downside is that they are more limited.

Existing undoable actions:
- `"set"` — set a property of a node in the editor to some value. Example:
  ```lua
  {
    action = "set",
    node_id = opts.selection,
    property = "text",
    value = "current time is " .. os.date()
  }
  ```
  `"set"` action requires these keys:
  - `node_id` — node id userdata. Alternatively, you can use resource path here instead of node id you received from the editor, for example `"/main/game.script"`;
  - `property` — a property of a node to set, e.g. `"text"`;
  - `value` — new value for a property. For `"text"` property it should be a string.

### Non-undoable actions

::: sidenote
Prefer using `editor.execute()`.
:::

Non-undoable action clears undo history, so if you want to undo such action, you will have to use other means, such as version control.

Existing non-undoable actions:
- `"shell"` — execute a shell script. Example:
  ```lua
  {
    action = "shell",
    command = {
      "./scripts/minify-json.sh",
      editor.get(opts.selection, "path"):sub(2) -- trim leading "/"
    }
  }
  ```
  `"shell"` action requires `command` key, which is an array of command and it's arguments.

### Mixing actions and side effects

You can mix undoable and non-undoable actions. Actions are executed sequentially, hence depending on an order of actions you will end up losing ability to undo parts of that command.

Instead of returning actions from functions that expect them, you can just read and write to files directly using `io.open()`. This will trigger a resource reload that will clear undo history.
