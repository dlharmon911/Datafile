# Allegro Datafile Addon

A powerful and flexible datafile management system for Allegro 5 that allows you to package and compress multiple game assets into a single file.

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/dlharmon911/Datafile)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
  - [Initialization](#initialization)
  - [Creating and Managing Datafiles](#creating-and-managing-datafiles)
  - [Adding Objects](#adding-objects)
  - [Loading Objects](#loading-objects)
  - [Custom Object Types](#custom-object-types)
- [Supported Object Types](#supported-object-types)
- [Building](#building)
- [Examples](#examples)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

The Allegro Datafile Addon is a header-only library that provides functionality for creating, loading, saving, and managing datafiles - compressed archives that can contain multiple game assets such as bitmaps, fonts, audio samples, text data, and custom data types.

This addon is inspired by the classic Allegro 4 datafile system but redesigned for Allegro 5 with modern C features and improved flexibility.

### Key Benefits

- **Single File Deployment**: Package all your game assets into one compressed file
- **Type Safety**: Strongly typed object system with runtime type checking
- **Extensibility**: Easy-to-use plugin system for custom object types
- **Compression**: Built-in zlib compression for smaller file sizes
- **Cross-Platform**: Works on all platforms supported by Allegro 5

---

## Features

### Core Features
- ✅ Load and save compressed datafiles
- ✅ Support for multiple asset types out of the box
- ✅ Automatic compression using zlib
- ✅ Type-safe object access
- ✅ Memory-efficient storage

### Supported Asset Types
- 🖼️ **Bitmaps** - Images in any format supported by Allegro (PNG, JPEG, BMP, etc.)
- 🎨 **Bitmap Arrays** - Sprite sheets and tile maps automatically subdivided
- 🔤 **Fonts** - Both bitmap fonts and TrueType fonts
- 🔊 **Audio Samples** - Sound effects and music in WAV, OGG, etc.
- 📝 **Text** - UTF-8 encoded strings
- 📦 **Raw Data** - Arbitrary binary data
- 📁 **Nested Datafiles** - Hierarchical organization

### Advanced Features
- 🔌 Plugin system for custom object types
- 📏 Automatic bitmap array generation from sprite sheets
- 🎯 Named object access
- 🔍 Type introspection
- ♻️ Automatic resource management

---

## Installation

### Prerequisites

- **Allegro 5.2+** with the following addons:
  - allegro5 (core)
  - allegro_font
  - allegro_ttf
  - allegro_image
  - allegro_audio
  - allegro_acodec
  - allegro_primitives
- **zlib** compression library
- **C99** or later compiler

### Header-Only Installation

This is a header-only library. Simply:

1. Copy `include/d_datafile.h` to your project's include directory
2. In **one** C/C++ file in your project, define the implementation:

```c
#define ALLEGRO_DATAFILE_IMPLENTATION
#include "d_datafile.h"
```

3. In all other files, just include the header normally:

```c
#include "d_datafile.h"
```

### Using with a Build System

#### CMake Example
```cmake
find_package(Allegro5 REQUIRED COMPONENTS main font ttf image audio acodec primitives)
find_package(ZLIB REQUIRED)

add_executable(mygame main.c)
target_link_libraries(mygame 
    ${ALLEGRO5_LIBRARIES}
    ${ZLIB_LIBRARIES}
)
target_include_directories(mygame PRIVATE include)
```

---

## Quick Start

### Basic Usage Example

```c
#define ALLEGRO_DATAFILE_IMPLENTATION
#include "d_datafile.h"

int main(int argc, char** argv)
{
    // Initialize Allegro
    al_init();
    al_init_image_addon();

    // Initialize the datafile addon
    if (!al_init_datafile_addon()) {
        fprintf(stderr, "Failed to initialize datafile addon\n");
        return -1;
    }

    // Create a new datafile
    ALLEGRO_DATAFILE* datafile = al_create_datafile();

    // Add a bitmap
    ALLEGRO_BITMAP* sprite = al_load_bitmap("hero.png");
    al_add_datafile_object(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP, 
                          "hero_sprite", sprite);

    // Add an audio sample
    ALLEGRO_SAMPLE* sound = al_load_sample("jump.wav");
    al_add_datafile_object(&datafile, ALLEGRO_DATAFILE_TYPE_SAMPLE,
                          "jump_sound", sound);

    // Save the datafile
    al_save_datafile("assets.dat", datafile);

    // Later, load it back
    ALLEGRO_DATAFILE* loaded = al_load_datafile("assets.dat");

    // Access objects by index
    size_t count = al_get_datafile_object_count(loaded);
    for (size_t i = 0; i < count; i++) {
        const char* name = al_get_datafile_object_name(loaded, i);
        printf("Object %zu: %s\n", i, name);

        // Access the data
        if (loaded[i].type == ALLEGRO_DATAFILE_TYPE_BITMAP) {
            ALLEGRO_BITMAP* bmp = (ALLEGRO_BITMAP*)loaded[i].data;
            al_draw_bitmap(bmp, 0, 0, 0);
        }
    }

    // Cleanup
    al_destroy_datafile(loaded);
    al_destroy_datafile(datafile);
    al_shutdown_datafile_addon();

    return 0;
}
```

---

## API Reference

### Initialization

#### `bool al_init_datafile_addon(void)`
Initializes the datafile addon and registers all default object types.

**Returns:** `true` on success, `false` on failure.

**Note:** Must be called after initializing Allegro and before using any other datafile functions.

#### `void al_shutdown_datafile_addon(void)`
Shuts down the addon and frees all resources.

#### `bool al_is_datafile_addon_initialized(void)`
Checks if the addon has been initialized.

**Returns:** `true` if initialized, `false` otherwise.

#### `uint32_t al_get_datafile_addon_version(void)`
Gets the version number of the addon.

**Returns:** Version as a 32-bit integer (e.g., 0x00010000 for version 1.0.0).

---

### Creating and Managing Datafiles

#### `ALLEGRO_DATAFILE* al_create_datafile(void)`
Creates a new empty datafile.

**Returns:** Pointer to the new datafile, or `NULL` on failure.

**Example:**
```c
ALLEGRO_DATAFILE* datafile = al_create_datafile();
if (!datafile) {
    fprintf(stderr, "Failed to create datafile\n");
}
```

#### `ALLEGRO_DATAFILE* al_load_datafile(const char* filename)`
Loads a datafile from disk. The file can be compressed or uncompressed.

**Parameters:**
- `filename` - Path to the datafile

**Returns:** Pointer to the loaded datafile, or `NULL` on failure.

**Example:**
```c
ALLEGRO_DATAFILE* datafile = al_load_datafile("game_assets.dat");
if (!datafile) {
    fprintf(stderr, "Failed to load datafile\n");
}
```

#### `int32_t al_save_datafile(const char* filename, const ALLEGRO_DATAFILE* datafile)`
Saves a datafile to disk with compression.

**Parameters:**
- `filename` - Path where the datafile should be saved
- `datafile` - Pointer to the datafile to save

**Returns:** `0` on success, `-1` on failure.

**Example:**
```c
if (al_save_datafile("game_assets.dat", datafile) != 0) {
    fprintf(stderr, "Failed to save datafile\n");
}
```

#### `void al_destroy_datafile(ALLEGRO_DATAFILE* datafile)`
Destroys a datafile and frees all associated resources.

**Parameters:**
- `datafile` - Pointer to the datafile to destroy

**Note:** This calls the appropriate destroy callback for each object in the datafile.

#### `size_t al_get_datafile_object_count(const ALLEGRO_DATAFILE* datafile)`
Gets the number of objects in a datafile.

**Parameters:**
- `datafile` - Pointer to the datafile

**Returns:** Number of objects, or `0` if datafile is `NULL`.

#### `const char* al_get_datafile_object_name(const ALLEGRO_DATAFILE* datafile, size_t index)`
Gets the name of an object by index.

**Parameters:**
- `datafile` - Pointer to the datafile
- `index` - Zero-based index of the object

**Returns:** Pointer to the object's name string, or `NULL` on failure.

**Example:**
```c
for (size_t i = 0; i < al_get_datafile_object_count(datafile); i++) {
    const char* name = al_get_datafile_object_name(datafile, i);
    uint32_t type = datafile[i].type;
    printf("Object %zu: %s (type: 0x%08X)\n", i, name, type);
}
```

---

### Adding Objects

#### `int32_t al_add_datafile_object(ALLEGRO_DATAFILE** datafile, uint32_t type, const char* name, void* data)`
Adds an object to a datafile.

**Parameters:**
- `datafile` - Pointer to pointer to the datafile (may be reallocated)
- `type` - Type identifier (e.g., `ALLEGRO_DATAFILE_TYPE_BITMAP`)
- `name` - Name/identifier for the object
- `data` - Pointer to the object data

**Returns:** `0` on success, `-1` on failure.

**Important:** Always pass the datafile as a pointer-to-pointer, as this function may reallocate the datafile structure.

**Example:**
```c
ALLEGRO_BITMAP* bmp = al_load_bitmap("sprite.png");
al_add_datafile_object(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP, 
                       "player_sprite", bmp);
```

#### `int32_t al_add_datafile_file_object(ALLEGRO_DATAFILE** datafile, uint32_t type, const char* name, const char* filename, ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC loader)`
Loads an object from a file and adds it to the datafile.

**Parameters:**
- `datafile` - Pointer to pointer to the datafile
- `type` - Type identifier
- `name` - Name for the object
- `filename` - Path to the file to load
- `loader` - Function to load the object

**Returns:** `0` on success, `-1` on failure.

**Example:**
```c
al_add_datafile_file_object(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP,
    "background", "bg.png",
    (ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC)al_load_datafile_bitmap_f);
```

#### `int32_t al_add_datafile_file_object_args(ALLEGRO_DATAFILE** datafile, uint32_t type, const char* name, const char* filename, ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC loader, const void* args)`
Loads an object from a file with additional arguments and adds it to the datafile.

**Parameters:**
- `datafile` - Pointer to pointer to the datafile
- `type` - Type identifier
- `name` - Name for the object
- `filename` - Path to the file to load
- `loader` - Function to load the object
- `args` - Additional arguments for the loader

**Returns:** `0` on success, `-1` on failure.

**Example:**
```c
ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA args = {".png", 32, 32};
al_add_datafile_file_object_args(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP_ARRAY,
    "tiles", "tileset.png",
    (ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC)al_load_datafile_bitmap_array_f,
    &args);
```

---

### Loading Objects

#### `ALLEGRO_BITMAP* al_load_datafile_bitmap_f(ALLEGRO_FILE* file, const char* identifier)`
Loads a bitmap from an open file handle.

**Parameters:**
- `file` - Open file handle
- `identifier` - Format identifier (e.g., ".png"), or `NULL` to auto-detect

**Returns:** Pointer to the bitmap, or `NULL` on failure.

#### `ALLEGRO_BITMAP_ARRAY* al_load_datafile_bitmap_array_f(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA* args)`
Loads a bitmap and divides it into an array of sub-bitmaps.

**Parameters:**
- `file` - Open file handle
- `args` - Configuration specifying grid dimensions

**Returns:** Pointer to the bitmap array, or `NULL` on failure.

**Example:**
```c
ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA config = {".png", 32, 32};
ALLEGRO_FILE* file = al_fopen("sprites.png", "rb");
ALLEGRO_BITMAP_ARRAY* sprites = al_load_datafile_bitmap_array_f(file, &config);
al_fclose(file);

// Draw individual sprites
al_draw_bitmap(sprites->sub_bitmap[0], x, y, 0);
```

#### `ALLEGRO_SAMPLE* al_load_datafile_sample_f(ALLEGRO_FILE* file, const char* identifier)`
Loads an audio sample from an open file handle.

**Parameters:**
- `file` - Open file handle
- `identifier` - Format identifier (e.g., ".wav"), or `NULL` to auto-detect

**Returns:** Pointer to the sample, or `NULL` on failure.

#### `ALLEGRO_FONT* al_load_datafile_ttf_font_f(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_TTF_FONT_DATA* args)`
Loads a TrueType font from an open file handle.

**Parameters:**
- `file` - Open file handle
- `args` - Font configuration (filename, size, flags)

**Returns:** Pointer to the font, or `NULL` on failure.

#### `ALLEGRO_FONT* al_load_datafile_bitmap_font_f(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_BITMAP_FONT_DATA* args)`
Loads a bitmap font from an open file handle.

**Parameters:**
- `file` - Open file handle
- `args` - Font configuration with character ranges

**Returns:** Pointer to the font, or `NULL` on failure.

#### `ALLEGRO_USTR* al_load_datafile_text_f(ALLEGRO_FILE* file)`
Loads a UTF-8 encoded text string.

**Parameters:**
- `file` - Open file handle

**Returns:** Pointer to the string, or `NULL` on failure.

#### `ALLEGRO_DATA* al_load_datafile_data_f(ALLEGRO_FILE* file)`
Loads raw binary data.

**Parameters:**
- `file` - Open file handle

**Returns:** Pointer to the data structure, or `NULL` on failure.

---

### Custom Object Types

#### `bool al_register_datafile_object_type(uint32_t type, const ALLEGRO_DATAFILE_TYPE_VTABLE* vtable)`
Registers a custom object type handler.

**Parameters:**
- `type` - Type identifier (use `AL_ID` macro)
- `vtable` - Virtual function table defining operations

**Returns:** `true` on success, `false` on failure.

**Example:**
```c
#define MY_CUSTOM_TYPE AL_ID('C','U','S','T')

static const char* my_type_name(void) {
    return "CUSTOM_TYPE";
}

static void* my_type_load(ALLEGRO_FILE* f) {
    MyObject* obj = malloc(sizeof(MyObject));
    al_fread(f, obj, sizeof(MyObject));
    return obj;
}

static int32_t my_type_save(ALLEGRO_FILE* f, void* data) {
    MyObject* obj = (MyObject*)data;
    return al_fwrite(f, obj, sizeof(MyObject)) == sizeof(MyObject) ? 0 : -1;
}

static int32_t my_type_destroy(void* data) {
    free(data);
    return 0;
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE my_vtable = {
    my_type_name,
    my_type_load,
    my_type_save,
    my_type_destroy,
    NULL  // render function (optional)
};

// Register the type
al_register_datafile_object_type(MY_CUSTOM_TYPE, &my_vtable);
```

---

## Supported Object Types

| Type | Constant | Description |
|------|----------|-------------|
| Bitmap | `ALLEGRO_DATAFILE_TYPE_BITMAP` | Single image/texture |
| Bitmap Array | `ALLEGRO_DATAFILE_TYPE_BITMAP_ARRAY` | Sprite sheet/tileset |
| Font | `ALLEGRO_DATAFILE_TYPE_FONT` | Bitmap or TrueType font |
| Sample | `ALLEGRO_DATAFILE_TYPE_SAMPLE` | Audio sample/sound effect |
| Text | `ALLEGRO_DATAFILE_TYPE_TEXT` | UTF-8 encoded string |
| Data | `ALLEGRO_DATAFILE_TYPE_DATA` | Raw binary data |
| File | `ALLEGRO_DATAFILE_TYPE_FILE` | Nested datafile |

---

## Building

### Windows (Visual Studio)

```bash
# Open the solution
start Datafile.sln

# Or build from command line
msbuild Datafile.sln /p:Configuration=Release
```

### Linux/Mac

```bash
# Using CMake
mkdir build
cd build
cmake ..
make

# Or manually
gcc -c main.c -o main.o -I./include
gcc main.o -o datafile_demo -lallegro -lallegro_font -lallegro_ttf \
    -lallegro_image -lallegro_audio -lallegro_acodec \
    -lallegro_primitives -lz
```

---

## Examples

### Complete Example: Creating a Game Asset Pack

```c
#define ALLEGRO_DATAFILE_IMPLENTATION
#include "d_datafile.h"

int main(void)
{
    // Initialize
    al_init();
    al_init_image_addon();
    al_init_audio_addon();
    al_init_acodec_addon();
    al_init_font_addon();
    al_init_ttf_addon();

    if (!al_init_datafile_addon()) {
        fprintf(stderr, "Failed to init datafile addon\n");
        return 1;
    }

    // Create datafile
    ALLEGRO_DATAFILE* pak = al_create_datafile();

    // Add sprites
    ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA sprite_config = {".png", 32, 32};
    al_add_datafile_file_object_args(&pak, 
        ALLEGRO_DATAFILE_TYPE_BITMAP_ARRAY,
        "player_sprites", "assets/player_sheet.png",
        (ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC)al_load_datafile_bitmap_array_f,
        &sprite_config);

    // Add sounds
    al_add_datafile_file_object(&pak,
        ALLEGRO_DATAFILE_TYPE_SAMPLE,
        "jump_sound", "assets/jump.wav",
        (ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC)al_load_datafile_sample_f);

    al_add_datafile_file_object(&pak,
        ALLEGRO_DATAFILE_TYPE_SAMPLE,
        "coin_sound", "assets/coin.wav",
        (ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC)al_load_datafile_sample_f);

    // Add font
    ALLEGRO_DATAFILE_TTF_FONT_DATA font_config = {"assets/font.ttf", 24, 0};
    al_add_datafile_file_object_args(&pak,
        ALLEGRO_DATAFILE_TYPE_FONT,
        "ui_font", "assets/font.ttf",
        (ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC)al_load_datafile_ttf_font_f,
        &font_config);

    // Save
    if (al_save_datafile("game.pak", pak) != 0) {
        fprintf(stderr, "Failed to save datafile\n");
        return 1;
    }

    printf("Created game.pak with %zu objects\n", 
           al_get_datafile_object_count(pak));

    // Cleanup
    al_destroy_datafile(pak);
    al_shutdown_datafile_addon();

    return 0;
}
```

### Loading and Using Assets

```c
int main(void)
{
    // Initialize...

    // Load the asset pack
    ALLEGRO_DATAFILE* pak = al_load_datafile("game.pak");

    if (!pak) {
        fprintf(stderr, "Failed to load game.pak\n");
        return 1;
    }

    // Find and use assets by name
    size_t count = al_get_datafile_object_count(pak);
    ALLEGRO_BITMAP_ARRAY* player_sprites = NULL;
    ALLEGRO_SAMPLE* jump_sound = NULL;
    ALLEGRO_FONT* font = NULL;

    for (size_t i = 0; i < count; i++) {
        const char* name = al_get_datafile_object_name(pak, i);

        if (strcmp(name, "player_sprites") == 0) {
            player_sprites = (ALLEGRO_BITMAP_ARRAY*)pak[i].data;
        }
        else if (strcmp(name, "jump_sound") == 0) {
            jump_sound = (ALLEGRO_SAMPLE*)pak[i].data;
        }
        else if (strcmp(name, "ui_font") == 0) {
            font = (ALLEGRO_FONT*)pak[i].data;
        }
    }

    // Use the assets
    if (player_sprites) {
        // Draw first sprite
        al_draw_bitmap(player_sprites->sub_bitmap[0], 100, 100, 0);
    }

    if (jump_sound) {
        // Play sound
        al_play_sample(jump_sound, 1.0, 0.0, 1.0, ALLEGRO_PLAYMODE_ONCE, NULL);
    }

    if (font) {
        // Draw text
        al_draw_text(font, al_map_rgb(255, 255, 255), 10, 10, 0, "Hello!");
    }

    // Cleanup
    al_destroy_datafile(pak);

    return 0;
}
```

---

## Contributing

Contributions are welcome! Please feel free to submit pull requests or open issues on GitHub.

### Development Setup

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Coding Style

- Follow the existing code style
- Use descriptive variable and function names
- Add documentation comments for public APIs
- Include examples for new features

---

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## Acknowledgments

- Inspired by the classic Allegro 4 datafile system
- Built on top of the excellent [Allegro 5](https://liballeg.org/) game programming library
- Uses [zlib](https://www.zlib.net/) for compression

---

## Links

- **GitHub Repository**: https://github.com/dlharmon911/Datafile
- **Allegro 5**: https://liballeg.org/
- **API Documentation**: [docs/API_Reference.md](docs/API_Reference.md)

---

*For detailed API documentation, see [docs/API_Reference.md](docs/API_Reference.md)*

---

### ALLEGRO_DATAFILE_TYPE_BITMAP_ARRAY
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_BITMAP_ARRAY = ((uint32_t)AL_ID('B', 'M', 'P', 'A'));
```
**Description:** Bitmap array object type identifier.

**Usage:** Use for sprite sheets or tile maps that are divided into a grid.

---

### ALLEGRO_DATAFILE_TYPE_FONT
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_FONT = ((uint32_t)AL_ID('F', 'O', 'N', 'T'));
```
**Description:** Font object type identifier.

**Usage:** Use for both bitmap fonts and TrueType fonts.

---

### ALLEGRO_DATAFILE_TYPE_SAMPLE
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_SAMPLE = ((uint32_t)AL_ID('S', 'A', 'M', 'P'));
```
**Description:** Audio sample object type identifier.

**Usage:** Use for sound effects and music stored as audio samples.

---

### ALLEGRO_DATAFILE_TYPE_FILE
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_FILE = ((uint32_t)AL_ID('F', 'I', 'L', 'E'));
```
**Description:** Nested datafile object type identifier.

**Usage:** Use for hierarchical datafile structures.

---

### ALLEGRO_DATAFILE_TYPE_DATA
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_DATA = ((uint32_t)AL_ID('D', 'A', 'T', 'A'));
```
**Description:** Raw data object type identifier.

**Usage:** Use for arbitrary binary data.

---

### ALLEGRO_DATAFILE_TYPE_TEXT
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_TEXT = ((uint32_t)AL_ID('T', 'E', 'X', 'T'));
```
**Description:** Text/string object type identifier.

**Usage:** Use for UTF-8 encoded text strings.

---

### ALLEGRO_DATAFILE_TYPE_UNDEFINED
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_UNDEFINED = ((uint32_t)-1);
```
**Description:** Undefined/invalid object type identifier.

**Usage:** Used to indicate an invalid or uninitialized type.

---

## Callback Function Types

Function pointer types used for implementing custom object type handlers.

### ALLEGRO_DATAFILE_TYPE_NAMER_FUNC
```c
typedef const char* (*ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)(void);
```
**Description:** Function pointer type for getting the name of a datafile object type.

**Returns:** Pointer to a null-terminated string containing the type name.

**Example:**
```c
const char* my_type_name(void) {
    return "MY_CUSTOM_TYPE";
}
```

---

### ALLEGRO_DATAFILE_TYPE_LOADER_FUNC
```c
typedef void* (*ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)(ALLEGRO_FILE* f);
```
**Description:** Function pointer type for loading a datafile object from a file.

**Parameters:**
- `f` - File handle to read from

**Returns:** Pointer to the loaded object, or NULL on failure.

**Example:**
```c
void* my_type_load(ALLEGRO_FILE* f) {
    MyType* obj = malloc(sizeof(MyType));
    al_fread(f, obj, sizeof(MyType));
    return obj;
}
```

---

### ALLEGRO_DATAFILE_TYPE_SAVER_FUNC
```c
typedef int32_t (*ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)(ALLEGRO_FILE* f, void* data);
```
**Description:** Function pointer type for saving a datafile object to a file.

**Parameters:**
- `f` - File handle to write to
- `data` - Pointer to the object to save

**Returns:** 0 on success, -1 on failure.

**Example:**
```c
int32_t my_type_save(ALLEGRO_FILE* f, void* data) {
    MyType* obj = (MyType*)data;
    return al_fwrite(f, obj, sizeof(MyType)) == sizeof(MyType) ? 0 : -1;
}
```

---

### ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC
```c
typedef int32_t (*ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)(void* data);
```
**Description:** Function pointer type for destroying/freeing a datafile object.

**Parameters:**
- `data` - Pointer to the object to destroy

**Returns:** 0 on success, -1 on failure.

**Example:**
```c
int32_t my_type_destroy(void* data) {
    MyType* obj = (MyType*)data;
    // Clean up any internal resources
    free(obj);
    return 0;
}
```

---

### ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC
```c
typedef int32_t (*ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)(void* data);
```
**Description:** Function pointer type for rendering a datafile object.

**Parameters:**
- `data` - Pointer to the object to render

**Returns:** 0 on success, -1 on failure.

---

### ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC
```c
typedef void* (*ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC)(ALLEGRO_FILE* file);
```
**Description:** Function pointer type for loading an object from a file without additional arguments.

**Parameters:**
- `file` - File handle to read from

**Returns:** Pointer to the loaded object, or NULL on failure.

---

### ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC
```c
typedef void* (*ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC)(ALLEGRO_FILE* file, const void* data);
```
**Description:** Function pointer type for loading an object from a file with additional arguments.

**Parameters:**
- `file` - File handle to read from
- `data` - Pointer to additional arguments/configuration data

**Returns:** Pointer to the loaded object, or NULL on failure.

---

## Data Structures

### ALLEGRO_DATAFILE_TYPE_VTABLE
```c
typedef struct ALLEGRO_DATAFILE_TYPE_VTABLE {
    ALLEGRO_DATAFILE_TYPE_NAMER_FUNC name;
    ALLEGRO_DATAFILE_TYPE_LOADER_FUNC load;
    ALLEGRO_DATAFILE_TYPE_SAVER_FUNC save;
    ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC destroy;
    ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC render;
} ALLEGRO_DATAFILE_TYPE_VTABLE;
```
**Description:** Virtual function table for datafile object type handlers. This structure defines the operations that can be performed on a specific datafile object type.

**Members:**
- `name` - Function to get the object name
- `load` - Function to load the object
- `save` - Function to save the object
- `destroy` - Function to destroy the object
- `render` - Function to render the object

**Example:**
```c
static const ALLEGRO_DATAFILE_TYPE_VTABLE my_vtable = {
    my_type_name,
    my_type_load,
    my_type_save,
    my_type_destroy,
    my_type_render
};
```

---

### ALLEGRO_DATA
```c
typedef struct ALLEGRO_DATA {
    size_t size;
    uint8_t data[];
} ALLEGRO_DATA;
```
**Description:** Container for raw binary data. This structure stores arbitrary binary data with a size field followed by the actual data bytes in a flexible array member.

**Members:**
- `size` - Size of the data in bytes
- `data` - Flexible array member containing the actual data

**Usage:**
```c
ALLEGRO_DATA* raw_data = al_load_datafile_data_f(file);
printf("Data size: %zu bytes\n", raw_data->size);
// Access data: raw_data->data[0], raw_data->data[1], etc.
```

---

### ALLEGRO_BITMAP_ARRAY
```c
typedef struct ALLEGRO_BITMAP_ARRAY {
    ALLEGRO_BITMAP* bitmap;
    ALLEGRO_BITMAP** sub_bitmap;
    size_t count;
    int32_t width;
    int32_t height;
} ALLEGRO_BITMAP_ARRAY;
```
**Description:** Container for an array of bitmap sub-regions. This structure represents a bitmap that has been subdivided into a grid of smaller sub-bitmaps, useful for sprite sheets or tile maps.

**Members:**
- `bitmap` - The source bitmap containing all sub-bitmaps
- `sub_bitmap` - Array of pointers to sub-bitmap regions
- `count` - Number of sub-bitmaps in the array
- `width` - Width of each sub-bitmap in pixels
- `height` - Height of each sub-bitmap in pixels

**Example:**
```c
ALLEGRO_BITMAP_ARRAY* sprites = /* load bitmap array */;
// Draw the 5th sprite
al_draw_bitmap(sprites->sub_bitmap[4], x, y, 0);
```

---

### ALLEGRO_DATAFILE
```c
typedef struct ALLEGRO_DATAFILE {
    void* data;
    uint32_t type;
} ALLEGRO_DATAFILE;
```
**Description:** Container for a single datafile object entry. This structure represents one object stored in a datafile, containing a pointer to the data and its type identifier.

**Members:**
- `data` - Pointer to the actual object data
- `type` - Type identifier (e.g., ALLEGRO_DATAFILE_TYPE_BITMAP)

**Example:**
```c
ALLEGRO_DATAFILE* datafile = al_load_datafile("assets.dat");
// Access first object
if (datafile[0].type == ALLEGRO_DATAFILE_TYPE_BITMAP) {
    ALLEGRO_BITMAP* bmp = (ALLEGRO_BITMAP*)datafile[0].data;
    al_draw_bitmap(bmp, 0, 0, 0);
}
```

---

## Core Functions

### al_get_datafile_addon_version
```c
uint32_t al_get_datafile_addon_version(void);
```
**Description:** Gets the version number of the datafile addon.

**Returns:** Version number as a 32-bit unsigned integer.

**Example:**
```c
uint32_t version = al_get_datafile_addon_version();
printf("Datafile addon version: 0x%08X\n", version);
```

---

### al_is_datafile_addon_initialized
```c
bool al_is_datafile_addon_initialized(void);
```
**Description:** Checks if the datafile addon has been initialized.

**Returns:** true if initialized, false otherwise.

**Example:**
```c
if (!al_is_datafile_addon_initialized()) {
    al_init_datafile_addon();
}
```

---

### al_init_datafile_addon
```c
bool al_init_datafile_addon(void);
```
**Description:** Initializes the datafile addon. This function must be called before using any other datafile functions. It initializes all required Allegro subsystems and registers default object types.

**Returns:** true on success, false on failure.

**Example:**
```c
if (!al_init_allegro(ALLEGRO_VERSION_INT)) {
    return -1;
}
if (!al_init_datafile_addon()) {
    fprintf(stderr, "Failed to initialize datafile addon\n");
    return -1;
}
```

---

### al_shutdown_datafile_addon
```c
void al_shutdown_datafile_addon(void);
```
**Description:** Shuts down the datafile addon and frees all resources. After calling this function, the addon must be reinitialized before use.

**Example:**
```c
al_shutdown_datafile_addon();
```

---

### al_register_datafile_object_type
```c
bool al_register_datafile_object_type(uint32_t type, const ALLEGRO_DATAFILE_TYPE_VTABLE* vtable);
```
**Description:** Registers a custom datafile object type handler. This allows you to add support for custom object types in datafiles.

**Parameters:**
- `type` - The type identifier for this object type (use AL_ID macro)
- `vtable` - Pointer to the virtual function table defining the type's operations

**Returns:** true on success, false on failure.

**Example:**
```c
#define MY_CUSTOM_TYPE AL_ID('M','Y','T','P')

static const ALLEGRO_DATAFILE_TYPE_VTABLE my_vtable = {
    my_object_name,
    my_type_load,
    my_type_save,
    my_type_destroy,
    my_type_render
};

al_register_datafile_object_type(MY_CUSTOM_TYPE, &my_vtable);
```

---

### al_create_datafile
```c
ALLEGRO_DATAFILE* al_create_datafile(void);
```
**Description:** Creates a new empty datafile.

**Returns:** Pointer to the new datafile, or NULL on failure.

**Example:**
```c
ALLEGRO_DATAFILE* datafile = al_create_datafile();
if (!datafile) {
    fprintf(stderr, "Failed to create datafile\n");
    return -1;
}
```

---

### al_load_datafile
```c
ALLEGRO_DATAFILE* al_load_datafile(const char* filename);
```
**Description:** Loads a datafile from disk. The file can be compressed or uncompressed. This function automatically handles decompression if needed.

**Parameters:**
- `filename` - Path to the datafile to load

**Returns:** Pointer to the loaded datafile, or NULL on failure.

**Example:**
```c
ALLEGRO_DATAFILE* datafile = al_load_datafile("assets.dat");
if (!datafile) {
    fprintf(stderr, "Failed to load datafile\n");
    return -1;
}
```

---

### al_destroy_datafile
```c
void al_destroy_datafile(ALLEGRO_DATAFILE* datafile);
```
**Description:** Destroys a datafile and frees all associated resources. This function will call the appropriate destroy callback for each object stored in the datafile.

**Parameters:**
- `datafile` - Pointer to the datafile to destroy

**Example:**
```c
al_destroy_datafile(datafile);
datafile = NULL;
```

---

### al_get_datafile_object_count
```c
size_t al_get_datafile_object_count(const ALLEGRO_DATAFILE* datafile);
```
**Description:** Gets the number of objects stored in a datafile.

**Parameters:**
- `datafile` - Pointer to the datafile

**Returns:** Number of objects in the datafile, or 0 if datafile is NULL.

**Example:**
```c
size_t count = al_get_datafile_object_count(datafile);
printf("Datafile contains %zu objects\n", count);
```

---

### al_get_datafile_object_name
```c
const char* al_get_datafile_object_name(const ALLEGRO_DATAFILE* datafile, size_t index);
```
**Description:** Gets the name of an object in the datafile by index.

**Parameters:**
- `datafile` - Pointer to the datafile
- `index` - Index of the object (0-based)

**Returns:** Pointer to the object's name string, or NULL on failure.

**Example:**
```c
for (size_t i = 0; i < al_get_datafile_object_count(datafile); i++) {
    const char* name = al_get_datafile_name(datafile, i);
    printf("Object %zu: %s\n", i, name);
}
```

---

### al_add_datafile_object
```c
int32_t al_add_datafile_object(ALLEGRO_DATAFILE** datafile, uint32_t type, const char* name, void* data);
```
**Description:** Adds an object to a datafile. Note: This function may reallocate the datafile structure, so always use the pointer passed by reference.

**Parameters:**
- `datafile` - Pointer to pointer to the datafile (may be modified)
- `type` - Type identifier for the object
- `name` - Name/identifier for the object
- `data` - Pointer to the object data

**Returns:** 0 on success, -1 on failure.

**Example:**
```c
ALLEGRO_BITMAP* bmp = al_load_bitmap("sprite.png");
if (al_add_datafile_object(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP, "hero_sprite", bmp) != 0) {
    fprintf(stderr, "Failed to add object\n");
}
```

---

### al_add_datafile_file_object
```c
int32_t al_add_datafile_file_object(ALLEGRO_DATAFILE** datafile, uint32_t type, 
                                     const char* name, const char* filename, 
                                     const ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC loader);
```
**Description:** Loads an object from a file and adds it to the datafile.

**Parameters:**
- `datafile` - Pointer to pointer to the datafile (may be modified)
- `type` - Type identifier for the object
- `name` - Name/identifier for the object
- `filename` - Path to the file containing the object
- `loader` - Function to use for loading the object

**Returns:** 0 on success, -1 on failure.

**Example:**
```c
al_add_datafile_file_object(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP, 
                            "background", "bg.png", 
                            (ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC)al_load_datafile_bitmap_f);
```

---

### al_add_datafile_file_object_args
```c
int32_t al_add_datafile_file_object_args(ALLEGRO_DATAFILE** datafile, uint32_t type, 
                                          const char* name, const char* filename, 
                                          const ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC loader, 
                                          const void* args);
```
**Description:** Loads an object from a file with additional arguments and adds it to the datafile.

**Parameters:**
- `datafile` - Pointer to pointer to the datafile (may be modified)
- `type` - Type identifier for the object
- `name` - Name/identifier for the object
- `filename` - Path to the file containing the object
- `loader` - Function to use for loading the object
- `args` - Pointer to additional arguments for the loader function

**Returns:** 0 on success, -1 on failure.

**Example:**
```c
ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA args = {".png", 32, 32};
al_add_datafile_file_object_args(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP_ARRAY,
                                  "tiles", "tileset.png",
                                  (ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC)al_load_datafile_bitmap_array_f,
                                  &args);
```

---

## Helper Structures

### ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA
```c
typedef struct ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA {
    const char* identifier;
    int32_t width;
    int32_t height;
} ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA;
```
**Description:** Configuration data for loading bitmap arrays. This structure specifies how to divide a source bitmap into a grid of sub-bitmaps.

**Members:**
- `identifier` - File format identifier (e.g., ".png"). May be NULL to auto-detect bitmap type.
- `width` - Width of each sub-bitmap cell
- `height` - Height of each sub-bitmap cell

**Example:**
```c
ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA config = {
    .identifier = ".png",
    .width = 32,
    .height = 32
};
```

---

### ALLEGRO_DATAFILE_BITMAP_FONT_DATA
```c
typedef struct ALLEGRO_DATAFILE_BITMAP_FONT_DATA {
    const char* identifier;
    int range_count;
    int ranges[];
} ALLEGRO_DATAFILE_BITMAP_FONT_DATA;
```
**Description:** Configuration data for loading bitmap fonts. This structure specifies the character ranges to extract from a bitmap font image.

**Members:**
- `identifier` - File format identifier (e.g., ".png"). May be NULL to auto-detect bitmap type.
- `range_count` - Number of character ranges
- `ranges` - Flexible array of range pairs (start, end)

**Example:**
```c
struct {
    const char* identifier;
    int range_count;
    int ranges[2];
} font_config = {
    .identifier = ".png",
    .range_count = 1,
    .ranges = {32, 127}  // ASCII printable characters
};
```

---

### ALLEGRO_DATAFILE_TTF_FONT_DATA
```c
typedef struct ALLEGRO_DATAFILE_TTF_FONT_DATA {
    const char* filename;
    int size;
    int flags;
} ALLEGRO_DATAFILE_TTF_FONT_DATA;
```
**Description:** Configuration data for loading TrueType fonts. This structure specifies the parameters for loading a TTF font file.

**Members:**
- `filename` - Path to the TTF font file
- `size` - Font size in pixels
- `flags` - Font loading flags (see Allegro font addon)

**Example:**
```c
ALLEGRO_DATAFILE_TTF_FONT_DATA config = {
    .filename = "arial.ttf",
    .size = 24,
    .flags = 0
};
```

---

## Object Loader Functions

### al_load_datafile_bitmap_f
```c
ALLEGRO_BITMAP* al_load_datafile_bitmap_f(ALLEGRO_FILE* file, const char* identifier);
```
**Description:** Loads a bitmap from an open file handle.

**Parameters:**
- `file` - Open file handle positioned at the bitmap data
- `identifier` - File format identifier (e.g., ".png", ".bmp"). May be NULL to auto-detect bitmap type.

**Returns:** Pointer to the loaded bitmap, or NULL on failure.

**Example:**
```c
ALLEGRO_FILE* file = al_fopen("image.png", "rb");
ALLEGRO_BITMAP* bmp = al_load_datafile_bitmap_f(file, ".png");
al_fclose(file);
```

---

### al_load_datafile_bitmap_array_f
```c
ALLEGRO_BITMAP_ARRAY* al_load_datafile_bitmap_array_f(ALLEGRO_FILE* file, 
                                                       const ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA* args);
```
**Description:** Loads a bitmap array from an open file handle. This function loads a bitmap and divides it into a grid of sub-bitmaps according to the parameters in the args structure.

**Parameters:**
- `file` - Open file handle positioned at the bitmap data
- `args` - Pointer to configuration data specifying grid dimensions

**Returns:** Pointer to the loaded bitmap array, or NULL on failure.

**Example:**
```c
ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA config = {".png", 32, 32};
ALLEGRO_FILE* file = al_fopen("sprites.png", "rb");
ALLEGRO_BITMAP_ARRAY* sprites = al_load_datafile_bitmap_array_f(file, &config);
al_fclose(file);
```

---

### al_load_datafile_sample_f
```c
ALLEGRO_SAMPLE* al_load_datafile_sample_f(ALLEGRO_FILE* file, const char* identifier);
```
**Description:** Loads an audio sample from an open file handle.

**Parameters:**
- `file` - Open file handle positioned at the sample data
- `identifier` - File format identifier (e.g., ".wav", ".ogg"). May be NULL to auto-detect sample type.

**Returns:** Pointer to the loaded sample, or NULL on failure.

**Example:**
```c
ALLEGRO_FILE* file = al_fopen("sound.wav", "rb");
ALLEGRO_SAMPLE* sample = al_load_datafile_sample_f(file, ".wav");
al_fclose(file);
```

---

### al_load_datafile_ttf_font_f
```c
ALLEGRO_FONT* al_load_datafile_ttf_font_f(ALLEGRO_FILE* file, 
                                          const ALLEGRO_DATAFILE_TTF_FONT_DATA* args);
```
**Description:** Loads a TrueType font from an open file handle.

**Parameters:**
- `file` - Open file handle positioned at the TTF font data
- `args` - Pointer to configuration data specifying font size and flags

**Returns:** Pointer to the loaded font, or NULL on failure.

**Example:**
```c
ALLEGRO_DATAFILE_TTF_FONT_DATA config = {"arial.ttf", 24, 0};
ALLEGRO_FILE* file = al_fopen("arial.ttf", "rb");
ALLEGRO_FONT* font = al_load_datafile_ttf_font_f(file, &config);
al_fclose(file);
```

---

### al_load_datafile_bitmap_font_f
```c
ALLEGRO_FONT* al_load_datafile_bitmap_font_f(ALLEGRO_FILE* file, 
                                             const ALLEGRO_DATAFILE_BITMAP_FONT_DATA* args);
```
**Description:** Loads a bitmap font from an open file handle. This function loads a bitmap containing font glyphs and extracts the specified character ranges.

**Parameters:**
- `file` - Open file handle positioned at the bitmap font data
- `args` - Pointer to configuration data specifying character ranges

**Returns:** Pointer to the loaded font, or NULL on failure.

**Example:**
```c
struct {
    const char* identifier;
    int range_count;
    int ranges[2];
} config = {".png", 1, {32, 127}};

ALLEGRO_FILE* file = al_fopen("font.png", "rb");
ALLEGRO_FONT* font = al_load_datafile_bitmap_font_f(file, 
                       (ALLEGRO_DATAFILE_BITMAP_FONT_DATA*)&config);
al_fclose(file);
```

---

### al_load_datafile_text_f
```c
ALLEGRO_USTR* al_load_datafile_text_f(ALLEGRO_FILE* file);
```
**Description:** Loads a text string from an open file handle. The text is loaded as a UTF-8 encoded string.

**Parameters:**
- `file` - Open file handle positioned at the text data

**Returns:** Pointer to the loaded string (ALLEGRO_USTR), or NULL on failure.

**Example:**
```c
ALLEGRO_FILE* file = al_fopen("text.txt", "rb");
ALLEGRO_USTR* text = al_load_datafile_text_f(file);
al_fclose(file);

if (text) {
    printf("Text: %s\n", al_cstr(text));
    al_ustr_free(text);
}
```

---

### al_load_datafile_data_f
```c
ALLEGRO_DATA* al_load_datafile_data_f(ALLEGRO_FILE* file);
```
**Description:** Loads raw binary data from an open file handle.

**Parameters:**
- `file` - Open file handle positioned at the data

**Returns:** Pointer to the loaded data structure, or NULL on failure.

**Example:**
```c
ALLEGRO_FILE* file = al_fopen("data.bin", "rb");
ALLEGRO_DATA* data = al_load_datafile_data_f(file);
al_fclose(file);

if (data) {
    printf("Loaded %zu bytes\n", data->size);
    // Access: data->data[0], data->data[1], etc.
    al_free(data);
}
```

---

## Complete Usage Example

```c
#include <allegro5/allegro5.h>
#include "d_datafile.h"

int main(int argc, char** argv) {
    // Initialize Allegro
    if (!al_init()) {
        fprintf(stderr, "Failed to initialize Allegro\n");
        return -1;
    }
    
    // Initialize datafile addon
    if (!al_init_datafile_addon()) {
        fprintf(stderr, "Failed to initialize datafile addon\n");
        return -1;
    }
    
    // Create a new datafile
    ALLEGRO_DATAFILE* datafile = al_create_datafile();
    
    // Add objects to the datafile
    ALLEGRO_BITMAP* bmp = al_load_bitmap("hero.png");
    al_add_datafile_object(&datafile, ALLEGRO_DATAFILE_TYPE_BITMAP, "hero", bmp);
    
    ALLEGRO_SAMPLE* sfx = al_load_sample("jump.wav");
    al_add_datafile_object(&datafile, ALLEGRO_DATAFILE_TYPE_SAMPLE, "jump_sound", sfx);
    
    // Save the datafile (implementation needed)
    // al_save_datafile(datafile, "game_assets.dat");
    
    // Load a datafile
    ALLEGRO_DATAFILE* loaded = al_load_datafile("game_assets.dat");
    
    if (loaded) {
        size_t count = al_get_datafile_object_count(loaded);
        printf("Loaded %zu objects\n", count);
        
        for (size_t i = 0; i < count; i++) {
            const char* name = al_get_datafile_name(loaded, i);
            printf("Object %zu: %s (type: 0x%08X)\n", i, name, loaded[i].type);
        }
        
        // Clean up
        al_destroy_datafile(loaded);
    }
    
    al_destroy_datafile(datafile);
    al_shutdown_datafile_addon();
    
    return 0;
}
```

---

## License

This documentation is provided as-is for the Allegro Datafile Addon.

---

*Generated from d_datafile.h comment directives*
