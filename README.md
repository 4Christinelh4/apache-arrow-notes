# Handbook

## 1. make an arrow::Table

## 2. basic operations for table
### 2.1 retrive all columns from row i to row i+j-1
```
// virtual std::shared_ptr<Table> Slice(int64_t offset, int64_t length) const = 0;
return table_->Slice(i, j);
```
### 2.2 work with columns: get the column by index
```
// virtual std::shared_ptr<ChunkedArray> column(int i) const = 0;
    
    std::cout << "\ncheck column\n";
    std::shared_ptr<arrow::ChunkedArray> get_other = tget->column(1);

    // normally num chunks = 1
    for (int j = 0; j < get_other->num_chunks(); ++j) {
        auto inner = get_other->chunk(j);
        auto double_arr = std::static_pointer_cast<arrow::DoubleArray>(inner);
            for(int m=0; m < double_arr->length(); ++m) {
                std::cout << double_arr->Value(j)<< std::endl;
            }
    }
    std::cout << std::endl;
```
### 2.3 concatenate many tables
tables need to share a same schema. Unpack the Result by `&std::shared_ptr<arrow::Table>`
```
// Result<std::shared_ptr<Table>> ConcatenateTables(
    // const std::vector<std::shared_ptr<Table>>& tables,
    // ConcatenateTablesOptions options = ConcatenateTablesOptions::Defaults(),
    // MemoryPool* memory_pool = default_memory_pool());

    auto res_concat = arrow::ConcatenateTables(add_tables
    , {.unify_schemas = true}).Value(result);
```
### 2.4 get all fields (name of columns) of a table
```
    std::vector<std::shared_ptr<arrow::Field>> fields_ = t0->fields();
    std::for_each(fields_.begin(), fields_.end(), [](const std::shared_ptr<arrow::Field> &f){
        std::cout << f->name() << std::endl;
    });
```

## 3. table and parquet files
### 3.1 table -> disk
```
    ARROW_ASSIGN_OR_RAISE(auto outfile, arrow::io::FileOutputStream::Open(PAR_NAME));
    PARQUET_THROW_NOT_OK(parquet::arrow::WriteTable(*shared_table
    , arrow::default_memory_pool(), outfile, CHUNK_SZ ));

    // CHUNK_SZ has a very large default value. It's optional
```
### 3.2 disk -> table (memory)
```
    std::shared_ptr<arrow::io::ReadableFile> infile;
   
    ARROW_ASSIGN_OR_RAISE(infile, arrow::io::ReadableFile::Open(PAR_NAME));

    std::unique_ptr<parquet::arrow::FileReader> reader;

    // Note that Parquet's OpenFile() takes the reader by reference, rather than returning
    // a reader.
    PARQUET_THROW_NOT_OK(
        parquet::arrow::OpenFile(infile, arrow::default_memory_pool(), &reader));

    std::shared_ptr<arrow::Table> parquet_table;
    // Read the table.
    PARQUET_THROW_NOT_OK(reader->ReadTable(&parquet_table));
```
reader: only read some rows into the table. 
Basically, one chunk is one "row group"
```
    for (int m = 0; m < reader->num_row_groups(); ++m) {
        PARQUET_THROW_NOT_OK(reader->RowGroup(m)->ReadTable(&table));
        std::cout << "Loaded " << table->num_rows() << " rows in " << table->num_columns()
                << " columns." << std::endl;

        auto res0 = arrow::PrettyPrint(*table, {}, &std::cout );    
    }
```

## 4. arrow ipc files
### 4.1 table -> disk
CHUNK_SZ is optional here.
```
    ARROW_ASSIGN_OR_RAISE(auto output_file
    , arrow::io::FileOutputStream::Open(ARROW_NAME));

    ARROW_ASSIGN_OR_RAISE(std::shared_ptr<arrow::ipc::RecordBatchWriter> batch_writer
    , arrow::ipc::MakeFileWriter(output_file,  shared_table->schema()));

#ifdef USE_CHUNK
    std::cout << "chunk used\n";
    ARROW_RETURN_NOT_OK(batch_writer->WriteTable(*shared_table, CHUNK_SZ));
#else
    ARROW_RETURN_NOT_OK(batch_writer->WriteTable(*shared_table));
#endif
    ARROW_RETURN_NOT_OK(batch_writer->Close());

```
### 4.2 disk -> table
normally, num_record_batches = 1
```
    ARROW_ASSIGN_OR_RAISE(auto in_file
    , arrow::io::ReadableFile::Open(ARROW_NAME));   

    ARROW_ASSIGN_OR_RAISE(auto ipc_reader
    , arrow::ipc::RecordBatchFileReader::Open(in_file));

    std::shared_ptr<arrow::Table> tb;
    std::vector<std::shared_ptr<arrow::RecordBatch>> batches;

    batches.reserve(ipc_reader->num_record_batches());

    // a_record: Result<std::shared_ptr<RecordBatch>>
    for(int i=0; i<ipc_reader->num_record_batches(); ++i) {
        ARROW_ASSIGN_OR_RAISE(auto a_record, ipc_reader->ReadRecordBatch(i));
        batches.emplace_back(std::move(a_record));
    }

    auto r1 = arrow::Table::FromRecordBatches(ipc_reader->schema()
    , batches ).Value(&tb); // assign value in tb
```
