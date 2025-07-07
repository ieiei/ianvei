``` rust
footer::parse_metadata()
```

ParquetMetaData
{
	file_metadata: FileMetaData,
	row_groups: Vec<RowGroupMetaData>
	page_indexes: Option<Vec<Vec<Index>>>
	offset_indexes: Option<Vec<Vec<Vec<PageLocation>>>>
}

FileMetaData
{
	version: i32,
	num_rows: i64,
	created_by: Option<String>
	key_value_metadata: Option<Vec<{Key: String, Value: Option<String>}>>
	schema_descr: Arc<SchemaDescriptor>
	column_orders: Option<Vec<ColumnOrder>>
}

RowGroupMetaData 
{
	columns: Vec<ColumnChunkMetaData>
	num_rows: i64
	sorting_columns: Option<Vec<SortingColumn>>
	total_byte_size: i64
	schema_descr: Arc<SchemaDescriptor>
	page_offset_index: Option<Vec<Vec<PageLocation>>>
}



pub struct SortingColumn {
  column_idx: i32,
  descending: bool,
  nulls_first: bool,
}


ColumnChunkMetaData
{
	column_type: Type,
	column_path: ColumnPath,
	column_descr: Arc<SchemaDescriptor>,
	encodings: Vec<Encoding>,
	file_path: Option<String>,
    file_offset: i64,
    num_values: i64,
    compression: Compression,
    total_compressed_size: i64,
    total_uncompressed_size: i64,
    data_page_offset: i64,
    index_page_offset: Option<i64>,
    dictionary_page_offset: Option<i64>,
    statistics: Option<Statistics>,
    encoding_stats: Option<Vec<PageEncodingStats>>,
    bloom_filter_offset: Option<i64>,
    offset_index_offset: Option<i64>,
    offset_index_length: Option<i32>,
    column_index_offset: Option<i64>,
    column_index_length: Option<i32>,
}

struct PageLocation {
  pub offset: i64,
  pub compressed_page_size: i32,
  pub first_row_index: i64,
}


enum Type {
    BOOLEAN,
    INT32,
    INT64,
    INT96,
    FLOAT,
    DOUBLE,
    BYTE_ARRAY,
    FIXED_LEN_BYTE_ARRAY,
}


ColumnDescriptor 
{
    primitive_type: Arc<Type>,
    max_def_level: i16,
    max_rep_level: i16,
    path: ColumnPath,
}

ColumnPath
{
	parts: Vec<String>,
}


enum Encoding {
    PLAIN,
    RLE,
    BIT_PACKED,
    DELTA_BINARY_PACKED,
    DELTA_LENGTH_BYTE_ARRAY,
    DELTA_BYTE_ARRAY,
    RLE_DICTIONARY,
    BYTE_STREAM_SPLIT,
}

enum Index {
    NONE,
    BOOLEAN(NativeIndex<bool>),
    INT32(NativeIndex<i32>),
    INT64(NativeIndex<i64>),
    INT96(NativeIndex<Int96>),
    FLOAT(NativeIndex<f32>),
    DOUBLE(NativeIndex<f64>),
    BYTE_ARRAY(NativeIndex<ByteArray>),
    FIXED_LEN_BYTE_ARRAY(NativeIndex<ByteArray>),
}

NativeIndex<T> {
    physical_type: Type,
    indexes: Vec<PageIndex<T>>,
    boundary_order: i32,
}

PageIndex<T> {
    min: Option<T>,
    max: Option<T>,
    null_count: Option<i64>,
}

SchemaDescriptor {
    schema: TypePtr,
    leaves: Vec<Arc<ColumnDescriptor>>,
    leaf_to_base: Vec<usize>,
}


PARGUET_MAGIC = [u8; 4] = [b'P', b'A', b'R', b'1'];  4bytes
PARQUET_FOOTER = [u8; 8]     8bytes


metadata_len = parquet_footer[..4]




