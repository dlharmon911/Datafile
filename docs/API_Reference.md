# Allegro Datafile Addon API Reference

Version: 1.0

## Table of Contents

- [Overview](#overview)
- [Datafile Object Types](#datafile-object-types)
- [Callback Function Types](#callback-function-types)
- [Data Structures](#data-structures)
- [Core Functions](#core-functions)
- [Helper Structures](#helper-structures)
- [Object Loader Functions](#object-loader-functions)

---

## Overview

The Allegro Datafile Addon provides functionality for creating, loading, saving, and managing datafiles - compressed archives that can contain multiple game assets such as bitmaps, fonts, audio samples, and custom data types.

### Features
- Support for multiple asset types (bitmaps, fonts, audio samples, text, raw data)
- Compressed storage using zlib
- Extensible type system for custom object types
- Bitmap arrays for sprite sheets and tile maps
- Both bitmap and TrueType font support

### Dependencies
- Allegro 5 core library
- Allegro font addon
- Allegro TTF addon
- Allegro image addon
- Allegro audio addon
- Allegro audio codec addon
- Allegro primitives addon
- zlib compression library

---

## Datafile Object Types

Pre-defined constants for identifying different object types stored in datafiles.

### ALLEGRO_DATAFILE_TYPE_BITMAP
```c
static const uint32_t ALLEGRO_DATAFILE_TYPE_BITMAP = ((uint32_t)AL_ID('B', 'M', 'P', ' '));
```
**Description:** Bitmap object type identifier.

**Usage:** Use this constant when adding or querying bitmap objects in a datafile.

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
