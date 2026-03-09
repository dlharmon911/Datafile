#ifdef _DEBUG
#define DO_LOG(format, ...) printf("Error: " format "\nFile: %s\nLine: %d\n", __VA_ARGS__, __FILE__, __LINE__)
#else
#define DO_LOG(format, ...)
#endif

ALLEGRO_DEBUG_CHANNEL("datafile")

static const uint32_t ALLEGRO_DATAFILE_VERSION_INT = 0x00010000;
static const size_t ALLEGRO_DATAFILE_INITIAL_CAPACITY = 8;
static const size_t ALLEGRO_DATAFILE_GROWTH_FACTOR = 2;
static const char* _DATA_BITMAP_IDENTIFIER = ".png";
static const char* _DATA_SAMPLE_IDENTIFIER = ".wav";
static const int32_t _DATA_MAX_FONT_RANGES = 256;
static const char _DATAFILE_SIGNATURE[sizeof(size_t)] = { 0x86, 0xcd, 0x80, 0xac, 0x6a, 0xc3, 0x77, 0x76 };

typedef struct ALLEGRO_DATAFILE_HEADER
{
	char signature[sizeof(size_t)];
	size_t count;
	size_t capacity;
	ALLEGRO_USTR** names;
} ALLEGRO_DATAFILE_HEADER;

typedef struct ALLEGRO_DATAFILE_TYPE_NODE
{
	ALLEGRO_DATAFILE_TYPE_VTABLE vtable;
	struct ALLEGRO_DATAFILE_TYPE_NODE* next;
	uint32_t type;
} ALLEGRO_DATAFILE_TYPE_NODE;

// PACKFILE section begin
static void* gz_fopen_wrapper(const char* path, const char* mode)
{
	void* gz = NULL;

	if (!path)
	{
		DO_LOG("Invalid file path");
		return NULL;
	}

	if (!mode)
	{
		DO_LOG("Invalid file mode");
		return NULL;
	}

	gz = (void*)gzopen(path, mode);

	return gz;
}

static bool gz_fclose_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return false;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return false;
	}

	return Z_OK == gzclose(gz);
}

static size_t gz_fread_wrapper(ALLEGRO_FILE* f_ptr, void* ptr, size_t size)
{
	gzFile gz = NULL;
	int32_t byte_count = 0;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return false;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return false;
	}

	byte_count = gzread(gz, ptr, (uint32_t)size);

	if (byte_count != size)
	{
		DO_LOG("Failed to read the expected number of bytes");
		return 0;
	}

	return byte_count;
}

static size_t gz_fwrite_wrapper(ALLEGRO_FILE* f_ptr, const void* ptr, size_t size)
{
	gzFile gz = NULL;
	int32_t byte_count = 0;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return false;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return false;
	}

	byte_count = gzwrite(gz, ptr, (uint32_t)size);

	if (byte_count != size)
	{
		DO_LOG("Failed to write the expected number of bytes");
		return 0;
	}

	return byte_count;
}

static bool gz_fflush_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;
	int32_t result = Z_OK;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return false;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return false;
	}

	result = gzflush(gz, Z_SYNC_FLUSH);

	return Z_OK == result;
}

static int64_t gz_ftell_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;
	int64_t position = -1;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return -1;
	}

	position = gztell(gz);

	return position;
}

static bool gz_fseek_wrapper(ALLEGRO_FILE* f_ptr, int64_t offset, int32_t whence)
{
	gzFile gz = NULL;
	int64_t result = Z_OK;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return false;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return false;
	}

	result = gzseek(gz, offset, whence);

	return (result != -1);
}

static bool gz_feof_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return false;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return false;
	}

	return gzeof(gz) != 0;
}

static int32_t gz_ferror_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;
	int32_t errnum = 0;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return -1;
	}

	gzerror(gz, &errnum);

	return errnum;
}

static const char* gz_ferrmsg_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return "Invalid file pointer";
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return "Invalid gzFile pointer";
	}

	return gzerror(gz, NULL);
}

static void gz_fclearerr_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return;
	}

	gzclearerr(gz);
}
static int32_t gz_fungetc_wrapper(ALLEGRO_FILE* f_ptr, int32_t c)
{
	gzFile gz = NULL;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return -1;
	}

	return gzungetc(c, gz);
}

static off_t gz_fsize_wrapper(ALLEGRO_FILE* f_ptr)
{
	gzFile gz = NULL;
	int64_t current_pos = -1;
	int64_t end_pos = -1;

	if (!f_ptr)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	gz = (gzFile)al_get_file_userdata(f_ptr);

	if (!gz)
	{
		DO_LOG("Invalid gzFile pointer");
		return -1;
	}

	current_pos = gztell(gz);

	if (current_pos == -1)
	{
		DO_LOG("Failed to get current position");
		return -1;
	}

	end_pos = gzseek(gz, 0, ALLEGRO_SEEK_END);

	if (end_pos == -1)
	{
		DO_LOG("Failed to get end position");
		return -1;
	}

	if (gzseek(gz, (long)current_pos, ALLEGRO_SEEK_SET) == -1)
	{
		DO_LOG("Failed to restore current position");
		return -1;
	}

	return (off_t)end_pos;
}

static void gz_set_interface(void)
{
	static const ALLEGRO_FILE_INTERFACE gz_interface =
	{
		gz_fopen_wrapper,
		gz_fclose_wrapper,
		gz_fread_wrapper,
		gz_fwrite_wrapper,
		gz_fflush_wrapper,
		gz_ftell_wrapper,
		gz_fseek_wrapper,
		gz_feof_wrapper,
		gz_ferror_wrapper,
		gz_ferrmsg_wrapper,
		gz_fclearerr_wrapper,
		gz_fungetc_wrapper,
		gz_fsize_wrapper
	};

	al_set_new_file_interface(&gz_interface);
}
// PACKFILE section end

#ifdef ALLEGRO_BIG_ENDIAN
static uint32_t _al_datafile_swap_uint32(uint32_t value)
{
	return ((value >> 24) & 0x000000FF) |
		((value >> 8) & 0x0000FF00) |
		((value << 8) & 0x00FF0000) |
		((value << 24) & 0xFF000000);
}

static size_t _al_datafile_swap_size(size_t value)
{
	return ((value >> 56) & 0x00000000000000FFULL) |
		((value >> 40) & 0x000000000000FF00ULL) |
		((value >> 24) & 0x0000000000FF0000ULL) |
		((value >> 8) & 0x00000000FF000000ULL) |
		((value << 8) & 0x000000FF00000000ULL) |
		((value << 24) & 0x0000FF0000000000ULL) |
		((value << 40) & 0x00FF000000000000ULL) |
		((value << 56) & 0xFF00000000000000ULL);
}
#endif

static int32_t _al_datafile_read_uint(ALLEGRO_FILE* file, void* value, size_t size)
{
	size_t bytes_read = 0;

	if (al_feof(file))
	{
		DO_LOG("End of file reached");
		return -1;
	}

	bytes_read = al_fread(file, value, size);

	if (bytes_read != size)
	{
		DO_LOG("Failed to read the expected number of bytes");
		return -1;
	}

#ifdef ALLEGRO_BIG_ENDIAN
	if (size == sizeof(uint32_t))
	{
		*(uint32_t*)value = _al_datafile_swap_uint32(*(uint32_t*)value);
	}
	else if (size == sizeof(size_t))
	{
		*(size_t*)value = _al_datafile_swap_size(*(size_t*)value);
	}
#endif

	return 0;
}

static int32_t _al_datafile_write_uint(ALLEGRO_FILE* file, const void* value, size_t size)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (value == NULL)
	{
		DO_LOG("Invalid value pointer");
		return -1;
	}

	if (size == sizeof(uint32_t))
	{
		uint32_t temp = *(const uint32_t*)value;

#ifdef ALLEGRO_BIG_ENDIAN
		temp = _al_datafile_swap_uint32(temp);
#endif

		if (al_fwrite(file, &temp, sizeof(uint32_t)) != sizeof(uint32_t))
		{
			DO_LOG("Failed to write uint32_t value");
			return -1;
		}

	}
	else if (size == sizeof(size_t))
	{
		size_t temp = *(const size_t*)value;

#ifdef ALLEGRO_BIG_ENDIAN
		temp = _al_datafile_swap_size(temp);
#endif

		if (al_fwrite(file, &temp, sizeof(size_t)) != sizeof(size_t))
		{
			DO_LOG("Failed to write size_t value");
			return -1;
		}
	}

	return 0;
}

static void _al_datafile_assert_signature(const ALLEGRO_DATAFILE_HEADER* header)
{
	if (header == NULL)
	{
		DO_LOG("Invalid header pointer");
		return;
	}

	for (size_t i = 0; i < sizeof(header->signature); ++i)
	{
		ALLEGRO_ASSERT(header->signature[i] == _DATAFILE_SIGNATURE[i]);
	}
}

static void _al_datafile_destroy_header(ALLEGRO_DATAFILE_HEADER* header)
{
	if (header == NULL)
	{
		DO_LOG("Invalid header pointer");
		return;
	}

	if (header->names)
	{
		for (size_t i = 0; i < header->count; ++i)
		{
			if (header->names + i)
			{
				al_ustr_free(header->names[i]);
			}
		}
		al_free(header->names);
		header->names = NULL;
	}

	al_free(header);
}

// String functions

static int32_t _al_datafile_write_string(ALLEGRO_FILE* file, const ALLEGRO_USTR* name)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (name == NULL)
	{
		DO_LOG("Invalid name pointer");
		return -1;
	}

	size_t str_size = al_ustr_size(name);
	const char* str_data = al_cstr(name);

	if (_al_datafile_write_uint(file, &str_size, sizeof(size_t)) != 0)
	{
		DO_LOG("Failed to write string size");
		return -1;
	}

	if (al_fwrite(file, str_data, str_size) != str_size)
	{
		DO_LOG("Failed to write string data");
		return -1;
	}

	return 0;
}

static int32_t _al_datafile_read_string(ALLEGRO_FILE* file, ALLEGRO_USTR** name)
{
	size_t str_size = 0;

	if (_al_datafile_read_uint(file, &str_size, sizeof(size_t)) != 0)
	{
		DO_LOG("Failed to read string size");
		return -1;
	}

	if (str_size == 0) // Arbitrary limit to prevent excessive memory allocation
	{
		DO_LOG("Invalid string size");
		return -1;
	}

	char* buffer = (char*)al_malloc(str_size);

	if (buffer == NULL)
	{
		DO_LOG("Failed to allocate memory for string buffer");
		return -1;
	}

	size_t bytes_read = al_fread(file, buffer, str_size);

	if (bytes_read != str_size)
	{
		DO_LOG("Failed to read string data");
		al_free(buffer);
		return -1;
	}

	(*name) = al_ustr_new_from_buffer(buffer, str_size);

	al_free(buffer);

	if ((*name) == NULL)
	{
		DO_LOG("Failed to create string from buffer");
		return -1;
	}

	return 0;
}

static ALLEGRO_DATAFILE_HEADER* _al_datafile_create_header(size_t capacity)
{
	size_t size = sizeof(ALLEGRO_DATAFILE_HEADER) + sizeof(ALLEGRO_DATAFILE) * capacity;
	ALLEGRO_DATAFILE_HEADER* header = (ALLEGRO_DATAFILE_HEADER*)al_malloc(size);

	if (header == NULL)
	{
		DO_LOG("Failed to allocate memory for datafile header");
		return NULL;
	}

	header->count = 0;
	header->capacity = capacity;
	header->names = NULL;

	for (size_t i = 0; i < sizeof(header->signature); ++i)
	{
		header->signature[i] = _DATAFILE_SIGNATURE[i];
	}

	for (size_t i = 0; i < header->capacity; ++i)
	{
		ALLEGRO_DATAFILE* object = (ALLEGRO_DATAFILE*)(header + 1) + i;
		object->data = NULL;
		object->type = ALLEGRO_DATAFILE_TYPE_UNDEFINED;
	}

	header->names = (ALLEGRO_USTR**)al_malloc(sizeof(ALLEGRO_USTR*) * capacity);

	if (header->names == NULL)
	{
		DO_LOG("Failed to allocate memory for datafile name array");
		al_free(header);
		header = NULL;
	}

	for (size_t i = 0; i < header->capacity; ++i)
	{
		header->names[i] = NULL;
	}

	return header;
}

static ALLEGRO_DATAFILE_HEADER* _al_resize_datafile_header(ALLEGRO_DATAFILE_HEADER* header)
{
	if (header == NULL)
	{
		DO_LOG("Invalid header pointer");
		return NULL;
	}

	size_t new_capacity = header->capacity * ALLEGRO_DATAFILE_GROWTH_FACTOR;
	size_t size = sizeof(ALLEGRO_DATAFILE_HEADER) + sizeof(ALLEGRO_DATAFILE) * new_capacity;

	ALLEGRO_DATAFILE_HEADER* new_header = (ALLEGRO_DATAFILE_HEADER*)al_realloc(header, size);

	if (new_header == NULL)
	{
		DO_LOG("Failed to reallocate memory for datafile header");
		return NULL;
	}

	for (size_t i = new_header->count; i < new_capacity; ++i)
	{
		ALLEGRO_DATAFILE* object = (ALLEGRO_DATAFILE*)(new_header + 1) + i;
		object->data = NULL;
		object->type = ALLEGRO_DATAFILE_TYPE_UNDEFINED;
	}

	ALLEGRO_USTR** new_names = (ALLEGRO_USTR**)al_realloc(new_header->names, sizeof(ALLEGRO_USTR*) * new_capacity);

	if (new_names == NULL)
	{
		DO_LOG("Failed to reallocate memory for datafile name array");
		al_free(new_header);
		return NULL;
	}

	for (size_t i = new_header->count; i < new_capacity; ++i)
	{
		new_names[i] = NULL;
	}

	new_header->names = new_names;
	new_header->capacity = new_capacity;

	return new_header;
}

// Bitmap data type functions

static const char* _al_datafile_bitmap_name(void)
{
	static const char* name = "BITMAP";

	return name;
}

static int32_t _al_datafile_bitmap_do_load(ALLEGRO_FILE* file, ALLEGRO_BITMAP** bitmap)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (bitmap == NULL)
	{
		DO_LOG("Invalid bitmap pointer");
		return -1;
	}

	(*bitmap) = al_load_bitmap_f(file, _DATA_BITMAP_IDENTIFIER);

	if (*bitmap == NULL)
	{
		DO_LOG("Failed to load bitmap");
		return -1;
	}

	return 0;
}

static void _al_datafile_bitmap_destroy(ALLEGRO_BITMAP* bitmap)
{
	if (bitmap != NULL)
	{
		al_destroy_bitmap(bitmap);
	}
}

static ALLEGRO_BITMAP* _al_datafile_bitmap_load(ALLEGRO_FILE* file)
{
	ALLEGRO_BITMAP* bitmap = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}

	if (bitmap == NULL)
	{
		DO_LOG("Invalid bitmap pointer");
		return NULL;
	}

	if (_al_datafile_bitmap_do_load(file, &bitmap) != 0)
	{
		DO_LOG("Failed to load bitmap");
		_al_datafile_bitmap_destroy(bitmap);
		bitmap = NULL;
	}

	return bitmap;
}

static int32_t  _al_datafile_bitmap_save(ALLEGRO_FILE* file, ALLEGRO_BITMAP* bitmap)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (bitmap == NULL)
	{
		DO_LOG("Invalid bitmap pointer");
		return -1;
	}

	if (!al_save_bitmap_f(file, _DATA_BITMAP_IDENTIFIER, bitmap))
	{
		DO_LOG("Failed to save bitmap");
		return -1;
	}

	return 0;
}

static void _al_datafile_bitmap_render(const ALLEGRO_BITMAP* font)
{
	if (font != NULL)
	{
		// This is a placeholder implementation. You can customize this function to render the bitmap as needed.
	}
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE* _al_datafile_bitmap_vtable(void)
{
	static const ALLEGRO_DATAFILE_TYPE_VTABLE vtable =
	{
		(ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)_al_datafile_bitmap_name,
		(ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)_al_datafile_bitmap_load,
		(ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)_al_datafile_bitmap_save,
		(ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)_al_datafile_bitmap_destroy,
		(ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)_al_datafile_bitmap_render
	};

	return &vtable;
}

// Bitmap array data type functions

static const char* _al_datafile_bitmap_array_name(void)
{
	static const char* name = "BITMAP_ARRAY";

	return name;
}

static bool _al_datafile_is_bitmap_empty(ALLEGRO_BITMAP* bitmap, int32_t width, int32_t height)
{
	if (bitmap == NULL)
	{
		DO_LOG("Invalid bitmap pointer");
		return true;
	}

	const ALLEGRO_LOCKED_REGION* locked = al_lock_bitmap(bitmap, ALLEGRO_PIXEL_FORMAT_ANY, ALLEGRO_LOCK_READONLY);

	if (!locked)
	{
		DO_LOG("Failed to lock bitmap");
		return true;
	}

	for (int32_t y = 0; y < height; y++)
	{
		for (int32_t x = 0; x < width; x++)
		{
			ALLEGRO_COLOR pixel = al_get_pixel(bitmap, x, y);

			if (pixel.r != 0 || pixel.g != 0 || pixel.b != 0 || pixel.a != 0)
			{
				return false;
			}
		}
	}

	al_unlock_bitmap(bitmap);

	return true;
}

static size_t _al_datafile_calculate_bitmap_array_count(ALLEGRO_BITMAP* bitmap, int32_t width, int32_t height)
{
	if (bitmap == NULL)
	{
		DO_LOG("Invalid bitmap pointer");
		return 0;
	}

	size_t count = 0;
	int32_t w = al_get_bitmap_width(bitmap);
	int32_t h = al_get_bitmap_height(bitmap);
	
	for (int32_t j = 0; j < h; j += height)
	{
		for (int32_t i = 0; i < w; i += width)
		{
			ALLEGRO_BITMAP* sub_bitmap = al_create_sub_bitmap(bitmap, i, j, width, height);
			if (!sub_bitmap)
			{
				continue;
			}
			if (!_al_datafile_is_bitmap_empty(sub_bitmap, width, height))
			{
				count++;
			}
			al_destroy_bitmap(sub_bitmap);
		}
	}
	return count;
}

static int32_t _al_datafile_create_bitmap_array(ALLEGRO_BITMAP_ARRAY** bitmap_array, ALLEGRO_BITMAP* bitmap, int32_t width, int32_t height)
{
	if (bitmap_array == NULL)
	{
		DO_LOG("Invalid bitmap array pointer");
		return -1;
	}

	if (bitmap == NULL)
	{
		DO_LOG("Invalid bitmap pointer");
		return -1;
	}

	(*bitmap_array) = (ALLEGRO_BITMAP_ARRAY*)al_malloc(sizeof(ALLEGRO_BITMAP_ARRAY));

	if (!(*bitmap_array))
	{
		DO_LOG("Failed to allocate memory for bitmap array");
		return -1;
	}

	(*bitmap_array)->bitmap = bitmap;
	(*bitmap_array)->count = _al_datafile_calculate_bitmap_array_count(bitmap, width, height);
	(*bitmap_array)->width = width;
	(*bitmap_array)->height = height;
	(*bitmap_array)->sub_bitmap = (ALLEGRO_BITMAP**)al_malloc(sizeof(ALLEGRO_BITMAP*) * (*bitmap_array)->count);

	if (!(*bitmap_array)->sub_bitmap)
	{
		DO_LOG("Failed to allocate memory for sub bitmaps");
		return -1;
	}

	int32_t w = al_get_bitmap_width(bitmap);
	int32_t h = al_get_bitmap_height(bitmap);
	size_t index = 0;

	for (int32_t j = 0; j < h; j += height)
	{
		for (int32_t i = 0; i < w; i += width)
		{
			(*bitmap_array)->sub_bitmap[index] = al_create_sub_bitmap(bitmap, i, j, width, height);

			if (!(*bitmap_array)->sub_bitmap[index])
			{
				DO_LOG("Failed to create sub bitmap");
				return -1;
			}

			++index;
		}
	}

	return 0;
}

static void _al_datafile_bitmap_array_destroy(ALLEGRO_BITMAP_ARRAY* bitmap_array)
{
	if (bitmap_array == NULL)
	{
		DO_LOG("Invalid bitmap array pointer");
		return;
	}

	if (bitmap_array->sub_bitmap)
	{
		for (size_t i = 0; i < bitmap_array->count; i++)
		{
			if (bitmap_array->sub_bitmap[i])
			{
				al_destroy_bitmap(bitmap_array->sub_bitmap[i]);
			}
		}

		al_free(bitmap_array->sub_bitmap);
	}

	if (bitmap_array->bitmap)
	{
		al_destroy_bitmap(bitmap_array->bitmap);
	}

	al_free(bitmap_array);
}

static int32_t _al_datafile_bitmap_array_do_load(ALLEGRO_FILE* file, ALLEGRO_BITMAP_ARRAY* bitmap_array)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}
	
	if (bitmap_array == NULL)
	{
		DO_LOG("Invalid bitmap array pointer");
		return -1;
	}

	bitmap_array->bitmap = _al_datafile_bitmap_load(file);

	if (bitmap_array->bitmap == NULL)
	{
		DO_LOG("Failed to load bitmap");
		return -1;
	}

	if (_al_datafile_read_uint(file, (uint32_t*)&bitmap_array->width, sizeof(uint32_t)) != 0)
	{
		DO_LOG("Failed to read bitmap array width");
		return -1;
	}

	if (_al_datafile_read_uint(file, (uint32_t*)&bitmap_array->height, sizeof(uint32_t)) != 0)
	{
		DO_LOG("Failed to read bitmap array height");
		return -1;
	}

	if (_al_datafile_create_bitmap_array(&bitmap_array, bitmap_array->bitmap, bitmap_array->width, bitmap_array->height) != 0)
	{
		DO_LOG("Failed to create bitmap array");
		return -1;
	}

	return 0;
}

static ALLEGRO_BITMAP_ARRAY* _al_datafile_bitmap_array_load(ALLEGRO_FILE* file)
{
	ALLEGRO_BITMAP_ARRAY* bitmap_array = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}

	bitmap_array = (ALLEGRO_BITMAP_ARRAY*)al_malloc(sizeof(ALLEGRO_BITMAP_ARRAY));

	if (bitmap_array == NULL)
	{
		DO_LOG("Failed to allocate memory for bitmap array");
		return NULL;
	}

	if (_al_datafile_bitmap_array_do_load(file, bitmap_array) != 0)
	{
		DO_LOG("Failed to load bitmap array");
		_al_datafile_bitmap_array_destroy(bitmap_array);
		bitmap_array = NULL;
	}

	return bitmap_array;
}

static int32_t  _al_datafile_bitmap_array_save(ALLEGRO_FILE* file, ALLEGRO_BITMAP_ARRAY* bitmap_array)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");	
		return -1;
	}

	if (bitmap_array == NULL)
	{
		DO_LOG("Invalid bitmap array pointer");
		return -1;
	}

	if (!al_save_bitmap_f(file, _DATA_BITMAP_IDENTIFIER, bitmap_array->bitmap))
	{
		DO_LOG("Failed to save bitmap");
		return -1;
	}

	if (_al_datafile_write_uint(file, (uint32_t*)&bitmap_array->width, sizeof(uint32_t)) != 0)
	{
		DO_LOG("Failed to write bitmap array width");
		return -1;
	}

	if (_al_datafile_write_uint(file, (uint32_t*)&bitmap_array->height, sizeof(uint32_t)) != 0)
	{
		DO_LOG("Failed to write bitmap array height");
		return -1;
	}

	return 0;
}

static void _al_datafile_bitmap_array_render(const ALLEGRO_BITMAP_ARRAY* font)
{
	if (font != NULL)
	{
		// This is a placeholder implementation. You can customize this function to render the bitmap array as needed.
	}
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE* _al_datafile_bitmap_array_vtable(void)
{
	static const ALLEGRO_DATAFILE_TYPE_VTABLE vtable =
	{
		(ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)_al_datafile_bitmap_array_name,
		(ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)_al_datafile_bitmap_array_load,
		(ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)_al_datafile_bitmap_array_save,
		(ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)_al_datafile_bitmap_array_destroy,
		(ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)_al_datafile_bitmap_array_render
	};

	return &vtable;
}

// Data type functions

static const char* _al_datafile_data_name(void)
{
	static const char* name = "DATA";

	return name;
}

static void _al_datafile_data_destroy(ALLEGRO_DATA* data)
{
	if (data != NULL)
	{
		al_free(data);
	}
}

static int32_t  _al_datafile_data_do_load(ALLEGRO_FILE* file, ALLEGRO_DATA** data)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (data == NULL)
	{
		DO_LOG("Invalid data pointer");
		return -1;
	}

	size_t data_size = 0;

	if (_al_datafile_read_uint(file, &data_size, sizeof(size_t)) != 0)
	{
		DO_LOG("Failed to read data size");
		return -1;
	}

	(*data) = (ALLEGRO_DATA*)al_malloc(sizeof(ALLEGRO_DATA) + data_size);

	if (*data == NULL)
	{
		DO_LOG("Failed to allocate memory for data");
		return -1;
	}

	(*data)->size = data_size;

	if (al_fread(file, (*data)->data, data_size) != data_size)
	{
		DO_LOG("Failed to read data");
		return -1;
	}

	return 0;
}

static ALLEGRO_DATA* _al_datafile_data_load(ALLEGRO_FILE* file)
{
	ALLEGRO_DATA* data = NULL;
	
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}
	
	if (_al_datafile_data_do_load(file, &data) != 0)
	{
		DO_LOG("Failed to load data");
		_al_datafile_data_destroy(data);
		data = NULL;
	}

	return data;
}

static int32_t  _al_datafile_data_save(ALLEGRO_FILE* file, const ALLEGRO_DATA* data)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (data == NULL)
	{
		DO_LOG("Invalid data pointer");
		return -1;
	}

	if (_al_datafile_write_uint(file, &data->size, sizeof(size_t)) != 0)
	{
		DO_LOG("Failed to write data size");
		return -1;
	}

	if (al_fwrite(file, data->data, data->size) != data->size)
	{
		DO_LOG("Failed to write data");
		return -1;
	}

	return 0;
}

static void _al_datafile_data_render(const ALLEGRO_DATA* data)
{
	if (data != NULL)
	{
		// This is a placeholder implementation. You can customize this function to render the data as needed.
	}
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE* _al_datafile_data_vtable()
{
	static const ALLEGRO_DATAFILE_TYPE_VTABLE vtable =
	{
		(ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)_al_datafile_data_name,
		(ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)_al_datafile_data_load,
		(ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)_al_datafile_data_save,
		(ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)_al_datafile_data_destroy,
		(ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)_al_datafile_data_render
	};

	return &vtable;
}

// Font data type functions

static const char* _al_datafile_font_name(void)
{
	static const char* name = "FONT";

	return name;
}

static void _d_get_font_data(const ALLEGRO_FONT* font, const int32_t* ranges, size_t* count, int32_t* width, int32_t* height)
{
	if (!font)
	{
		DO_LOG("Invalid font pointer");
		return;
	}

	if (!count)
	{
		DO_LOG("Invalid count pointer");
		return;
	}

	if (!width)
	{
		DO_LOG("Invalid width pointer");
		return;
	}

	if (!height)
	{
		DO_LOG("Invalid height pointer");
		return;
	}

	*count = 0;
	*width = 0;
	*height = al_get_font_line_height(font);

	size_t range_max = ((size_t)_DATA_MAX_FONT_RANGES) << 1;

	for (size_t i = 0; i < range_max; i += 2)
	{
		int32_t start = ranges[i];
		int32_t end = ranges[i + 1];

		if (start == 0 && end == 0)
		{
			break;
		}

		for (int32_t c = start; c <= end; ++c)
		{
			int32_t char_width = al_get_glyph_width(font, c);

			if (char_width > *width)
			{
				*width = char_width;
			}

			++(*count);
		}
	}
}

static void _d_draw_glyph_to_bitmap(const ALLEGRO_FONT* font, const ALLEGRO_BITMAP* bitmap, int32_t char_code, int32_t x, int32_t y)
{
	if (!font)
	{
		DO_LOG("Invalid font pointer");
		return;
	}

	if (!bitmap)
	{
		DO_LOG("Invalid bitmap pointer");
		return;
	}

	ALLEGRO_USTR* str = al_ustr_newf("%c", char_code);

	if (!str)
	{
		DO_LOG("Failed to create string");
		return;
	}

	int32_t char_width = al_get_glyph_width(font, char_code);
	int32_t char_height = al_get_font_line_height(font);

	al_draw_filled_rectangle((float)x, (float)y, (float)(x + char_width), (float)(y + char_height), al_map_rgb(255, 0, 255));

	al_draw_ustr(font, al_map_rgb(255, 255, 255), (float)(x), (float)(y), 0, str);

	al_ustr_free(str);
}

static void _d_draw_glyph_range_to_bitmap(const ALLEGRO_FONT* font, const ALLEGRO_BITMAP* bitmap, int32_t* index, const int32_t range[2], int32_t chunk_width, int32_t chunk_height)
{
	if (!font)
	{
		DO_LOG("Invalid font pointer");
		return;
	}

	if (!bitmap)
	{
		DO_LOG("Invalid bitmap pointer");
		return;
	}

	if (!index)
	{
		DO_LOG("Invalid index pointer");
		return;
	}

	for (int32_t c = range[0]; c <= range[1]; ++c)
	{
		int32_t x = (*index % 16) * chunk_width;
		int32_t y = (*index / 16) * chunk_height;

		_d_draw_glyph_to_bitmap(font, bitmap, c, x + 1, y + 1);
		++(*index);
	}
}

static ALLEGRO_BITMAP* _d_convert_to_bitmap(ALLEGRO_FONT* font)
{
	ALLEGRO_BITMAP* bitmap = NULL;
	int32_t ranges[512] = { 0 };
	int32_t range_count = 0;
	int32_t char_width = 0;
	int32_t char_height = 0;
	int32_t chunk_width = 0;
	int32_t chunk_height = 0;
	size_t count = 0;
	int32_t index = 0;

	if (!font)
	{
		DO_LOG("Invalid font pointer");
		return NULL;
	}

	range_count = al_get_font_ranges(font, _DATA_MAX_FONT_RANGES, ranges);
	_d_get_font_data(font, ranges, &count, &char_width, &char_height);

	chunk_width = (char_width + 0x10) & 0xfffffff0;
	chunk_height = (char_height + 0x10) & 0xfffffff0;

	int32_t bitmap_width = 1 + (chunk_width << 4);
	int32_t bitmap_height = 1 + (chunk_height * (int32_t)(count >> 4));

	bitmap = al_create_bitmap(bitmap_width, bitmap_height);
	if (!bitmap)
	{
		DO_LOG("Failed to create bitmap");
		return NULL;
	}

	ALLEGRO_BITMAP* previous_target = al_get_target_bitmap();
	al_set_target_bitmap(bitmap);
	al_clear_to_color(al_map_rgb(255, 255, 0));

	for (int32_t i = 0; i < range_count; i += 2)
	{
		int32_t  char_range[2] = { ranges[i], ranges[i + 1] };
		_d_draw_glyph_range_to_bitmap(font, bitmap, &index, char_range, chunk_width, chunk_height);
	}

	al_set_target_bitmap(previous_target);

	return bitmap;
}

static void _al_datafile_font_destroy(ALLEGRO_FONT* font)
{
	if (font != NULL)
	{
		al_destroy_font(font);
	}
}

static int32_t _al_datafile_font_do_load(ALLEGRO_FILE* file, ALLEGRO_FONT** font)
{
	int32_t  ranges[] = { 0x0020, 0x007f };
	ALLEGRO_BITMAP* bitmap = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (font == NULL)
	{
		DO_LOG("Invalid font pointer");
		return -1;
	}

	bitmap = _al_datafile_bitmap_load(file);

	if (!bitmap)
	{
		DO_LOG("Failed to load bitmap");
		return -1;
	}

	ALLEGRO_COLOR mask_color = al_get_pixel(bitmap, 1, 1);

	al_convert_mask_to_alpha(bitmap, mask_color);

	(*font) = al_grab_font_from_bitmap(bitmap, 1, ranges);

	if (*font == NULL)
	{
		DO_LOG("Failed to grab font from bitmap");
		return -1;
	}

	return 0;
}

static ALLEGRO_FONT* _al_datafile_font_load(ALLEGRO_FILE* file)
{
	ALLEGRO_FONT* font = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}

	if (_al_datafile_font_do_load(file, &font) != 0)
	{
		DO_LOG("Failed to load font");
		_al_datafile_font_destroy(font);
		font = NULL;
	}

	return font;
}

static int32_t  _al_datafile_font_save(ALLEGRO_FILE* file, ALLEGRO_FONT* font)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (font == NULL)
	{
		DO_LOG("Invalid font pointer");
		return -1;
	}

	ALLEGRO_BITMAP* bitmap = _d_convert_to_bitmap(font);

	if (!bitmap)
	{
		DO_LOG("Failed to convert font to bitmap");
		return -1;
	}

	if (_al_datafile_bitmap_save(file, bitmap) != 0)
	{
		DO_LOG("Failed to save bitmap");
		return -1;
	}

	return 0;
}

static void _al_datafile_font_render(const ALLEGRO_FONT* font)
{
	if (font != NULL)
	{
		// This is a placeholder implementation. You can customize this function to render the font as needed.
	}
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE* _al_datafile_font_vtable()
{
	static const ALLEGRO_DATAFILE_TYPE_VTABLE vtable =
	{
		(ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)_al_datafile_font_name,
		(ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)_al_datafile_font_load,
		(ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)_al_datafile_font_save,
		(ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)_al_datafile_font_destroy,
		(ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)_al_datafile_font_render
	};

	return &vtable;
}

// Sample data type functions

static const char* _al_datafile_sample_name(void)
{
	static const char* name = "SAMPLE";

	return name;
}

static void _al_datafile_sample_destroy(ALLEGRO_SAMPLE* sample)
{
	if (sample != NULL)
	{
		al_destroy_sample(sample);
	}
}

static int32_t  _al_datafile_sample_do_load(ALLEGRO_FILE* file, ALLEGRO_SAMPLE** sample)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (sample == NULL)
	{
		DO_LOG("Invalid sample pointer");
		return -1;
	}

	(*sample) = al_load_sample_f(file, _DATA_SAMPLE_IDENTIFIER);

	if (*sample == NULL)
	{
		DO_LOG("Failed to load sample");
		return -1;
	}

	return 0;
}

static ALLEGRO_SAMPLE* _al_datafile_sample_load(ALLEGRO_FILE* file)
{
	ALLEGRO_SAMPLE* sample = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}

	if (_al_datafile_sample_do_load(file, &sample) != 0)
	{
		DO_LOG("Failed to load sample");
		_al_datafile_sample_destroy(sample);
		sample = NULL;
	}

	return sample;
}

static int32_t  _al_datafile_sample_save(ALLEGRO_FILE* file, ALLEGRO_SAMPLE* sample)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (sample == NULL)
	{
		DO_LOG("Invalid sample pointer");
		return -1;
	}

	if (!al_save_sample_f(file, _DATA_SAMPLE_IDENTIFIER, sample))
	{
		DO_LOG("Failed to save sample");
		return -1;
	}

	return 0;
}

static void _al_datafile_sample_render(const ALLEGRO_SAMPLE* sample)
{
	if (sample != NULL)
	{
		// This is a placeholder implementation. You can customize this function to render the sample as needed.
	}
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE* _al_datafile_sample_vtable()
{
	static const ALLEGRO_DATAFILE_TYPE_VTABLE vtable =
	{
		(ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)_al_datafile_sample_name,
		(ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)_al_datafile_sample_load,
		(ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)_al_datafile_sample_save,
		(ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)_al_datafile_sample_destroy,
		(ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)_al_datafile_sample_render
	};

	return &vtable;
}

// Text data type functions

static const char* _al_datafile_text_name(void)
{
	static const char* name = "TEXT";

	return name;
}

static void _al_datafile_text_destroy(ALLEGRO_USTR* text)
{
	if (text != NULL)
	{
		al_ustr_free(text);
	}
}

static int32_t  _al_datafile_text_do_load(ALLEGRO_FILE* file, ALLEGRO_USTR** text)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (text == NULL)
	{
		DO_LOG("Invalid text pointer");
		return -1;
	}

	if (_al_datafile_read_string(file, text) != 0)
	{
		DO_LOG("Failed to read text");
		return -1;
	}

	return 0;
}

static ALLEGRO_USTR* _al_datafile_text_load(ALLEGRO_FILE* file)
{
	ALLEGRO_USTR* text = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}

	if (_al_datafile_text_do_load(file, &text) != 0)
	{
		DO_LOG("Failed to load text");
		_al_datafile_text_destroy(text);
		text = NULL;
	}

	return text;
}

static int32_t  _al_datafile_text_save(ALLEGRO_FILE* file, const ALLEGRO_USTR* text)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (text == NULL)
	{
		DO_LOG("Invalid text pointer");
		return -1;
	}

	if (_al_datafile_write_string(file, text) != 0)
	{
		DO_LOG("Failed to write text");
		return -1;
	}

	return 0;
}

static void _al_datafile_text_render(const ALLEGRO_USTR* text)
{
	if (text != NULL)
	{
		// This is a placeholder implementation. You can customize this function to render the text as needed.
	}
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE* _al_datafile_text_vtable()
{
	static const ALLEGRO_DATAFILE_TYPE_VTABLE vtable =
	{
		(ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)_al_datafile_text_name,
		(ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)_al_datafile_text_load,
		(ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)_al_datafile_text_save,
		(ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)_al_datafile_text_destroy,
		(ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)_al_datafile_text_render
	};

	return &vtable;
}

// File data type functions

static const char* _al_datafile_file_name(void)
{
	static const char* name = "DATAFILE";

	return name;
}

static ALLEGRO_DATAFILE_TYPE_NODE* _al_datafile_type_head_node = NULL;

static ALLEGRO_DATAFILE_TYPE_NODE* _al_datafile_type_find_node(uint32_t type)
{
	ALLEGRO_DATAFILE_TYPE_NODE* current = _al_datafile_type_head_node;

	while (current)
	{
		if (current->type == type)
		{
			return current;
		}

		current = current->next;
	}

	return NULL;
}

static int32_t _al_datafile_read_data(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_HEADER* header, ALLEGRO_DATAFILE* entries)
{
	for (size_t i = 0; i < header->count; ++i)
	{
		ALLEGRO_DATAFILE* entry = (entries + i);
		ALLEGRO_USTR* name = header->names[i];

		if (_al_datafile_read_uint(file, &entry->type, sizeof(uint32_t)) != 0)
		{
			DO_LOG("Failed to read entry type");
			return -1;
		}

		if (_al_datafile_read_string(file, &name) != 0)
		{
			DO_LOG("Failed to read entry name");
			return -1;
		}

		const ALLEGRO_DATAFILE_TYPE_NODE* node = _al_datafile_type_find_node(entry->type);

		if (!node || !node->vtable.load)
		{
			DO_LOG("Failed to find datafile type node or load function");
			return -1;
		}

		entry->data = node->vtable.load(file);
		
		if (!entry->data)
		{
			DO_LOG("Failed to load entry data");
			return -1;
		}
	}

	return 0;
}

static void _al_datafile_file_destroy(ALLEGRO_DATAFILE* datafile)
{
	if (datafile == NULL)
	{
		DO_LOG("Invalid datafile pointer");
		return;
	}

	ALLEGRO_DATAFILE_HEADER* header = (ALLEGRO_DATAFILE_HEADER*)((char*)datafile - sizeof(ALLEGRO_DATAFILE_HEADER));
	_al_datafile_assert_signature(header);

	_al_datafile_destroy_header(header);
}

static int32_t  _al_datafile_file_do_load(ALLEGRO_FILE* file, ALLEGRO_DATAFILE** datafile)
{
	ALLEGRO_DATAFILE_HEADER* header = NULL;
	size_t count = 0;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (datafile == NULL)
	{
		DO_LOG("Invalid datafile pointer");
		return -1;
	}

	(*datafile) = NULL;

	if (_al_datafile_read_uint(file, &count, sizeof(size_t)) != 0)
	{
		DO_LOG("Failed to read datafile count");
		return -1;
	}

	header = _al_datafile_create_header(count);

	if (!header)
	{
		DO_LOG("Failed to create datafile header");
		return -1; // Memory allocation failed
	}

	if (_al_datafile_read_data(file, header, (ALLEGRO_DATAFILE*)(header + 1)) != 0)
	{
		DO_LOG("Failed to read datafile entries");
		_al_datafile_destroy_header(header);
		return -1; // Failed to load data entries
	}

	return 0;
}

static ALLEGRO_DATAFILE* _al_datafile_file_load(ALLEGRO_FILE* file)
{
	ALLEGRO_DATAFILE* datafile = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}

	if (_al_datafile_file_do_load(file, &datafile) != 0)
	{
		DO_LOG("Failed to load datafile");
		_al_datafile_file_destroy(datafile);
		datafile = NULL;
	}

	return datafile;
}

static int32_t _al_datafile_write_data(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_HEADER* header, const ALLEGRO_DATAFILE* entries)
{
	for (size_t i = 0; i < header->count; ++i)
	{
		const ALLEGRO_DATAFILE* entry = (entries + i);
		const ALLEGRO_USTR* name = header->names[i];

		if (_al_datafile_write_uint(file, &entry->type, sizeof(uint32_t)) != 0)
		{
			DO_LOG("Failed to write entry type");
			return -1;
		}

		if (_al_datafile_write_string(file, name) != 0)
		{
			DO_LOG("Failed to write entry name");
			return -1;
		}

		const ALLEGRO_DATAFILE_TYPE_NODE* node = _al_datafile_type_find_node(entry->type);

		if (!node || !node->vtable.save)
		{
			DO_LOG("Failed to find datafile type node or save function");
			return -1;
		}

		if (node->vtable.save(file, entry->data) != 0)
		{
			DO_LOG("Failed to save entry data");
			return -1;
		}
	}

	return 0;
}

static int32_t  _al_datafile_file_save(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE* datafile)
{
	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return -1;
	}

	if (datafile == NULL)
	{
		DO_LOG("Invalid datafile pointer");
		return -1;
	}

	const ALLEGRO_DATAFILE_HEADER* header = (const ALLEGRO_DATAFILE_HEADER*)((const char*)datafile - sizeof(ALLEGRO_DATAFILE_HEADER));
	_al_datafile_assert_signature(header);

	if (_al_datafile_write_uint(file, &header->count, sizeof(size_t)) != 0)
	{
		DO_LOG("Failed to write datafile header.");
		return -1;
	}

	if (_al_datafile_write_data(file, header, (const ALLEGRO_DATAFILE*)(header + 1)) != 0)
	{
		DO_LOG("Failed to write datafile entries.");
		return -1;
	}

	return 0;
}

static void _al_datafile_file_render(const ALLEGRO_DATAFILE* datafile)
{
	if (datafile != NULL)
	{
		// This is a placeholder implementation. You can customize this function to render the datafile as needed.
	}
}

static const ALLEGRO_DATAFILE_TYPE_VTABLE* _al_datafile_file_vtable()
{
	static const ALLEGRO_DATAFILE_TYPE_VTABLE vtable =
	{
		(ALLEGRO_DATAFILE_TYPE_NAMER_FUNC)_al_datafile_file_name,
		(ALLEGRO_DATAFILE_TYPE_LOADER_FUNC)_al_datafile_file_load,
		(ALLEGRO_DATAFILE_TYPE_SAVER_FUNC)_al_datafile_file_save,
		(ALLEGRO_DATAFILE_TYPE_DESTROYER_FUNC)_al_datafile_file_destroy,
		(ALLEGRO_DATAFILE_TYPE_RENDERER_FUNC)_al_datafile_file_render
	};

	return &vtable;
}

static ALLEGRO_DATAFILE_TYPE_NODE* _al_datafile_type_create_node(uint32_t type, const ALLEGRO_DATAFILE_TYPE_VTABLE* vtable)
{
	ALLEGRO_DATAFILE_TYPE_NODE* node = NULL;

	if (!vtable)
	{
		DO_LOG("VTable pointer is null.");
		return NULL;
	}

	node = (ALLEGRO_DATAFILE_TYPE_NODE*)al_malloc(sizeof(ALLEGRO_DATAFILE_TYPE_NODE));

	if (!node)
	{
		DO_LOG("Failed to allocate memory for datafile type node.");
		return NULL;
	}

	node->type = type;
	node->vtable.name = vtable->name;
	node->vtable.load = vtable->load;
	node->vtable.save = vtable->save;
	node->vtable.destroy = vtable->destroy;
	node->next = NULL;
	return node;
}


bool al_register_datafile_object_type(uint32_t type, const ALLEGRO_DATAFILE_TYPE_VTABLE* vtable)
{
	ALLEGRO_DATAFILE_TYPE_NODE* node = NULL;

	if (!vtable)
	{
		DO_LOG("VTable pointer is null.");
		return false;
	}

	node = _al_datafile_type_find_node(type);

	if (!node)
	{
		node = _al_datafile_type_create_node(type, vtable);

		if (!node)
		{
			DO_LOG("Failed to create datafile type node.");
			return false;
		}

		node->next = _al_datafile_type_head_node;

		_al_datafile_type_head_node = node;
	}

	node->vtable.name = vtable->name;
	node->vtable.load = vtable->load;
	node->vtable.save = vtable->save;
	node->vtable.destroy = vtable->destroy;

	return true;
}

uint32_t al_get_datafile_addon_version(void)
{
	return ALLEGRO_DATAFILE_VERSION_INT;
}

void al_shutdown_datafile_addon(void)
{
	ALLEGRO_DATAFILE_TYPE_NODE* current = _al_datafile_type_head_node;
	ALLEGRO_DATAFILE_TYPE_NODE* next = NULL;

	while (current)
	{
		next = current->next;
		al_free(current);
		current = next;
	}

	_al_datafile_type_head_node = NULL;
}

bool al_is_datafile_addon_initialized(void)
{
	return _al_datafile_type_head_node != NULL;
}

bool al_init_datafile_addon(void)
{
	if (!al_is_system_installed())
	{
		DO_LOG("Allegro system is not installed.");
		return false;
	}

	if (!al_is_audio_installed() && !al_install_audio())
	{
		DO_LOG("Failed to install audio.");
		return false;
	}

	if (!al_is_acodec_addon_initialized() && !al_init_acodec_addon())
	{
		DO_LOG("Failed to initialize acodec addon.");
		return false;
	}

	if (!al_is_image_addon_initialized() && !al_init_image_addon())
	{
		DO_LOG("Failed to initialize image addon.");
		return false;
	}

	if (!al_is_font_addon_initialized() && !al_init_font_addon())
	{
		DO_LOG("Failed to initialize font addon.");
		return false;
	}

	if (!al_is_ttf_addon_initialized() && !al_init_ttf_addon())
	{
		DO_LOG("Failed to initialize ttf addon.");
		return false;
	}

	if (!al_is_primitives_addon_initialized() && !al_init_primitives_addon())
	{
		DO_LOG("Failed to initialize primitives addon.");
		return false;
	}

	al_register_datafile_object_type(ALLEGRO_DATAFILE_TYPE_FILE, _al_datafile_file_vtable());
	al_register_datafile_object_type(ALLEGRO_DATAFILE_TYPE_BITMAP, _al_datafile_bitmap_vtable());
	al_register_datafile_object_type(ALLEGRO_DATAFILE_TYPE_BITMAP_ARRAY, _al_datafile_bitmap_array_vtable());
	al_register_datafile_object_type(ALLEGRO_DATAFILE_TYPE_FONT, _al_datafile_font_vtable());
	al_register_datafile_object_type(ALLEGRO_DATAFILE_TYPE_SAMPLE, _al_datafile_sample_vtable());
	al_register_datafile_object_type(ALLEGRO_DATAFILE_TYPE_TEXT, _al_datafile_text_vtable());
	al_register_datafile_object_type(ALLEGRO_DATAFILE_TYPE_DATA, _al_datafile_data_vtable());

	atexit(al_shutdown_datafile_addon);

	return true;
}

ALLEGRO_DATAFILE* al_create_datafile(void)
{
	ALLEGRO_DATAFILE_HEADER* header = _al_datafile_create_header(ALLEGRO_DATAFILE_INITIAL_CAPACITY);

	if (!header)
	{
		DO_LOG("Failed to create datafile header.");
		return NULL;
	}

	return (ALLEGRO_DATAFILE*)(header + 1);
}

ALLEGRO_DATAFILE* al_load_datafile(const char* filename)
{
	const ALLEGRO_FILE_INTERFACE* previous_interface = NULL;
	ALLEGRO_FILE* file = NULL;
	ALLEGRO_DATAFILE* datafile = NULL;

	if (filename == NULL)
	{
		DO_LOG("Filename pointer is null.");
		return NULL;
	}

	previous_interface = al_get_new_file_interface();
	gz_set_interface();

	file = al_fopen(filename, "rb");

	if (file)
	{
		datafile = _al_datafile_file_load(file);
	}

	al_fclose(file);

	al_set_new_file_interface(previous_interface);

	return datafile;
}

void al_destroy_datafile(ALLEGRO_DATAFILE* datafile)
{
	if (datafile)
	{
		ALLEGRO_DATAFILE_HEADER* header = (ALLEGRO_DATAFILE_HEADER*)((char*)datafile - sizeof(ALLEGRO_DATAFILE_HEADER));
		_al_datafile_assert_signature(header);
		_al_datafile_destroy_header(header);
	}
}

size_t al_get_datafile_object_count(const ALLEGRO_DATAFILE* datafile)
{
	if (!datafile)
	{
		DO_LOG("Datafile pointer is null.");
		return 0;
	}

	const ALLEGRO_DATAFILE_HEADER* header = (const ALLEGRO_DATAFILE_HEADER*)((const char*)datafile - sizeof(ALLEGRO_DATAFILE_HEADER));
	_al_datafile_assert_signature(header);
	return header->count;
}

const char* al_get_datafile_name(const ALLEGRO_DATAFILE* datafile, size_t index)
{
	if (!datafile)
	{
		DO_LOG("Datafile pointer is null.");
		return NULL;
	}
	
	const ALLEGRO_DATAFILE_HEADER* header = (const ALLEGRO_DATAFILE_HEADER*)((const char*)datafile - sizeof(ALLEGRO_DATAFILE_HEADER));
	
	_al_datafile_assert_signature(header);
	
	if (index >= header->count)
	{
		DO_LOG("Index out of bounds.");
		return NULL;
	}

	return al_cstr(header->names[index]);
}

int32_t al_add_datafile_object(ALLEGRO_DATAFILE** datafile, uint32_t type, const char* name, void* data)
{
	if (!datafile)
	{
		DO_LOG("Datafile pointer is null.");
		return -1;
	}

	if (!name)
	{
		DO_LOG("Name pointer is null.");
		return -1;
	}

	if (!data)
	{
		DO_LOG("Data pointer is null.");
		return -1;
	}

	ALLEGRO_DATAFILE_HEADER* header = (ALLEGRO_DATAFILE_HEADER*)((char*)datafile - sizeof(ALLEGRO_DATAFILE_HEADER));
	_al_datafile_assert_signature(header);

	if (header->count >= header->capacity)
	{
		header = _al_resize_datafile_header(header);

		if (!header)
		{
			DO_LOG("Failed to resize datafile header.");
			return -1; // Memory allocation failed
		}

		*datafile = (ALLEGRO_DATAFILE*)(header + 1);
	}

	size_t index = header->count;
	ALLEGRO_DATAFILE* object = (ALLEGRO_DATAFILE*)(header + 1) + index;
	ALLEGRO_USTR** object_name = (header->names + index);

	header->count++;

	object->type = type;
	object->data = data;
	*object_name = al_ustr_new(name);

	if (!*object_name)
	{
		DO_LOG("Failed to create object name string.");
		return -1; // Memory allocation failed
	}

	return 0;
}

int32_t al_add_datafile_file_object_args(ALLEGRO_DATAFILE** datafile, uint32_t type, const char* name, const char* filename, const ALLEGRO_DATAFILE_OBJECT_LOADER_ARGS_FUNC loader, const void* args)
{
	ALLEGRO_FILE* file = NULL;
	void* data = NULL;

	if (!datafile)
	{
		DO_LOG("Datafile pointer is null.");
		return -1;
	}

	if (!name)
	{
		DO_LOG("Name pointer is null.");
		return -1;
	}

	if (!filename)
	{
		DO_LOG("Filename pointer is null.");
		return -1;
	}

	file = al_fopen(filename, "rb");

	if (!file)
	{
		DO_LOG("Failed to open file: %s", filename);
		return -1; // Failed to open file
	}

	data = loader(file, args);

	al_fclose(file);

	if (!data)
	{
		DO_LOG("Failed to load data from file: %s", filename);
		return -1; // Failed to load data
	}

	return al_add_datafile_object(datafile, type, name, data);
}


int32_t al_add_datafile_file_object(ALLEGRO_DATAFILE** datafile, uint32_t type, const char* name, const char* filename, const ALLEGRO_DATAFILE_OBJECT_LOADER_FUNC loader)
{
	ALLEGRO_FILE* file = NULL;
	void* data = NULL;

	if (!datafile)
	{
		DO_LOG("Datafile pointer is null.");
		return -1;
	}

	if (!name)
	{
		DO_LOG("Name pointer is null.");
		return -1;
	}

	if (!filename)
	{
		DO_LOG("Filename pointer is null.");
		return -1;
	}

	file = al_fopen(filename, "rb");

	if (!file)
	{
		DO_LOG("Failed to open file: %s", filename);
		return -1; // Failed to open file
	}

	data = loader(file);

	al_fclose(file);

	if (!data)
	{
		DO_LOG("Failed to load data from file: %s", filename);
		return -1; // Failed to load data
	}

	return al_add_datafile_object(datafile, type, name, data);
}

ALLEGRO_BITMAP* al_load_datafile_bitmap_f(ALLEGRO_FILE* file, const char* identifier)
{
	ALLEGRO_BITMAP* bitmap = NULL;

	if (file == NULL)
	{
		DO_LOG("File pointer is null.");
		return NULL;
	}

	if (identifier)
	{
		bitmap = al_load_bitmap_f(file, identifier);
	}
	else
	{
		const char* file_identifier = al_identify_bitmap_f(file);

		if (file_identifier == NULL)
		{
			DO_LOG("File is not a valid bitmap.");
			return NULL;
		}

		bitmap = al_load_bitmap_f(file, file_identifier);
	}

	if (bitmap == NULL)
	{
		DO_LOG("Failed to load bitmap from file.");
		return NULL; // Failed to load bitmap
	}

	return bitmap;
}

ALLEGRO_BITMAP_ARRAY* al_load_datafile_bitmap_array_f(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_BITMAP_ARRAY_DATA* args)
{
	ALLEGRO_BITMAP_ARRAY* bitmap_array = NULL;
	ALLEGRO_BITMAP* bitmap = NULL;

	if (file == NULL)
	{
		DO_LOG("Invalid file pointer");
		return NULL;
	}

	if (args == NULL)
	{
		DO_LOG("Invalid arguments pointer");
		return NULL;
	}

	bitmap = al_load_datafile_bitmap_f(file, args->identifier);

	if (bitmap == NULL)
	{
		DO_LOG("Failed to load bitmap");
		return NULL;
	}

	if (_al_datafile_create_bitmap_array(&bitmap_array, bitmap, args->width, args->height) != 0)
	{
		DO_LOG("Failed to create bitmap array");
		_al_datafile_bitmap_array_destroy(bitmap_array);
		return NULL;
	}

	return bitmap_array;
}

ALLEGRO_SAMPLE* al_load_datafile_sample_f(ALLEGRO_FILE* file, const char* identifier)
{
	ALLEGRO_SAMPLE* sample = NULL;

	if (file == NULL)
	{
		DO_LOG("File pointer is null.");
		return NULL;
	}
	
	if (identifier)
	{
		sample = al_load_sample_f(file, identifier);
	}
	else
	{
		const char* file_identifier = al_identify_sample_f(file);
		if (file_identifier == NULL)
		{
			DO_LOG("File is not a valid sample.");
			return NULL;
		}
		sample = al_load_sample_f(file, file_identifier);
	}
	
	if (sample == NULL)
	{
		DO_LOG("Failed to load sample from file.");
		return NULL; // Failed to load sample
	}
	
	return sample;
}

ALLEGRO_FONT* al_load_datafile_bitmap_font_f(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_BITMAP_FONT_DATA* args)
{
	ALLEGRO_FONT* font = NULL;
	ALLEGRO_BITMAP* bitmap = NULL;

	if (file == NULL)
	{
		DO_LOG("Filename pointer is null.");
		return NULL;
	}

	if (args == NULL)
	{
		DO_LOG("Arguments pointer is null.");
		return NULL;
	}

	bitmap = al_load_datafile_bitmap_f(file, args->identifier);

	if (bitmap == NULL)
	{
		DO_LOG("Failed to load bitmap for font.");
		return NULL;
	}

	font = al_grab_font_from_bitmap(bitmap, args->range_count, args->ranges);

	if (font == NULL)
	{
		DO_LOG("Failed to load bitmap font from file.");
		return NULL; // Failed to load bitmap font
	}

	return font;
}

ALLEGRO_FONT* al_load_datafile_ttf_font_f(ALLEGRO_FILE* file, const ALLEGRO_DATAFILE_TTF_FONT_DATA* args)
{
	ALLEGRO_FONT* font = NULL;

	if (file == NULL)
	{
		DO_LOG("File pointer is null.");
		return NULL;
	}
	
	if (args == NULL)
	{
		DO_LOG("Font data pointer is null.");
		return NULL;
	}

	if (args->filename == NULL)
	{
		DO_LOG("Font filename pointer is null.");
		return NULL;
	}

	font = al_load_ttf_font_f(file, args->filename, args->size, args->flags);
	
	if (!font)
	{
		DO_LOG("Failed to load TTF font from file.");
		return NULL; // Failed to load TTF font
	}

	return font;
}

ALLEGRO_USTR* al_load_datafile_text_f(ALLEGRO_FILE* file)
{
	ALLEGRO_DATA* data = NULL;

	if (file == NULL)
	{
		DO_LOG("File pointer is null.");
		return NULL;
	}

	data = al_load_datafile_data_f(file);

	if (!data)
	{
		DO_LOG("Failed to load data from file.");
		return NULL;
	}

	ALLEGRO_USTR* text = al_ustr_new_from_buffer(data->data, data->size);

	al_free(data);

	if (!text)
	{
		DO_LOG("Failed to create text string from data.");
		return NULL; // Failed to create text string
	}

	return text;
}

ALLEGRO_DATA* al_load_datafile_data_f(ALLEGRO_FILE* file)
{
	ALLEGRO_DATA* data = NULL;

	if (file == NULL)
	{
		DO_LOG("File pointer is null.");
		return NULL;
	}

	// Get the file size
	int64_t file_size = al_fsize(file);

	if (file_size <= 0)
	{
		DO_LOG("Failed to get file size.");
		al_fclose(file);
		return NULL;
	}

	// Allocate memory for the data
	data = (ALLEGRO_DATA*)al_malloc(sizeof(ALLEGRO_DATA) + file_size);

	if (!data)
	{
		DO_LOG("Failed to allocate memory for data.");
		return NULL;
	}

	data->size = file_size;

	// Read the file into the data buffer
	if (al_fread(file, data->data, file_size) != file_size)
	{
		DO_LOG("Failed to read file.");
		al_free(data);
		return NULL;
	}

	al_fclose(file);

	return data;
}
