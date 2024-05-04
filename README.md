# ZipLib #
ZipLib is a lightweight C++11 library for working with ZIP archives with ease. The very useful feature of ZipLib is that **it is built around STL streams, no additional dependencies (like boost) are required**.

## Features ##
* Compression/decompresion using the DEFLATE algorithm
* Storing/fetching files without compression
* Adding, editing and removing files and directories in the archive
* Support of the data descriptor
* Data flow is driven via STL streams, therefore there is no need to load huge amount of data into the memory
* Support of PKWare encryption
* Support of entry comments and archive comments
* Contains very usable stream classes in *ZipLib/streams* like memory stream, substream or teestream
* Contains functions for easy (de)serialization (*ZipLib/streams/serialization.h*)
* Project is using only C++11 standard libraries, therefore there IS NO dependency on 3rd party libraries (except of included zlib, which uses standard libraries of C afterall :))
* Built around STL smart pointers
* Support of Windows and Linux

## Example of usage ##

### Adding, extracting and removing file using ZipFile wrapper ###
```C++
const char* zipFilename = "archive.zip";
ZipFile::AddFile(zipFilename, "file.txt");

// there is no need to create folders in the archive separately, because they are part of the filename
ZipFile::AddEncryptedFile(zipFilename, "fileOnDisk.txt", "destination/in/archive.txt", "password");

ZipFile::ExtractFile(zipFilename, "file.txt", "newFileOnDisk.txt");

ZipFile::ExtractEncryptedFile(zipFilename, "destination/in/archive.txt", "encrypted.txt", "password");

// because folders are part of the filename (until they are not created separately),
// the "destination" and "in" folders are deleted too
ZipFile::RemoveEntry(zipFilename, "destination/in/archive.txt");
```


### Very basic sample of a file compression ###
```C++
ZipArchive::Ptr archive = ZipFile::Open("archive.zip");

ZipArchiveEntry::Ptr entry = archive->CreateEntry("file.dat");

// if entry is nullptr, it means the file already exists in the archive
assert(entry != nullptr); 

std::ifstream contentStream("file.dat", std::ios::binary); //should not be destroyed before archive is written
entry->SetCompressionStream(contentStream);

// you may of course save it somewhere else
// note: the contentStream is pumped when the archive is saved
// if you want to read the stream before saving of the archive,
// it is need to set Immediate mode in SetCompressionStream method (see below)
ZipFile::SaveAndClose(archive, "archive.zip");


// If you want to use std::ostream interface for custom purposes
// such as sending data over network, where no seek() operation possible
// or you want to remove unnecessary seek operations on HDD
// mark entry to use data descriptor for pure streamed operation
// see https://en.wikipedia.org/wiki/Zip_(file_format)#File_headers
// entry->UseDataDescriptor(true); 
// archive->WriteToStream(std::ostream);
```


### Very basic sample of a file decompression ###
```C++
ZipArchive::Ptr archive = ZipFile::Open("archive.zip");

ZipArchiveEntry::Ptr entry = archive->GetEntry("file.txt");

// if the entry is password protected, it is necessary
// to set the password before getting a decompression stream
if (entry->IsPasswordProtected())
{
  // when decompressing an encrypted entry
  // there is no need to specify the use of data descriptor
  // (ZibLib will deduce if the data descriptor was used)
  entry->SetPassword("pass");
}

// if the entry is password protected and the provided password is wrong
// (or none is provided) the return value will be nullptr
std::istream* decompressStream = entry->GetDecompressionStream();

std::string line;
std::getline(*decompressStream, line);

printf("First line of a file: '%s'\n", line.c_str());
```


### Compressing a file with immediate mode ###
```C++
auto archive = ZipFile::Open("archive.zip");

auto entry = archive->CreateEntry("file.dat");

// if entry is nullptr, it means the file already exists in the archive
assert(entry != nullptr); 

{
  std::ifstream contentStream("file.dat", std::ios::binary); 
  entry->SetCompressionStream(   // data from contentStream are pumped here (into the memory)
    contentStream,
    ZipArchiveEntry::CompressionLevel::Default,
    ZipArchiveEntry::CompressionMethod::Deflate,
    ZipArchiveEntry::CompressionMode::Immediate
  );

  // the contentStream is destroyed here
}

ZipFile::SaveAndClose(archive, "archive.zip");
```

### Compress and encrypt the memory stream ###
```C++
char buffer[] = "A content to encrypt";
imemstream contentStream(buffer); // imemstream is located in Source/ZipLib/streams

auto archive = ZipFile::Open("archive.zip");
auto entry = archive->CreateEntry("file.dat");

entry->SetPassword("pass");
entry->UseDataDescriptor(); // read stream only once
entry->SetCompressionStream(contentStream);

ZipFile::SaveAndClose(archive, "archive.zip");
```

*You may find more samples in the **Source/Sample/Main.cpp** file.*

## Description of compression ##
You may specify several options how the input stream will be stored in the archive.

### Compression method ###
Second (optional) parameter of the ZipArchiveEntry::SetCompressionStream method. Default value is **Deflate**.

* **Deflate** - a standard compression algorithm used for ZIP archives
* **Stored** - no compression

### Compression level ###
Third (optional) parameter of the ZipArchiveEntry::SetCompressionStream method. Default value is **Default**.

* **Stored** - data from the input stream will not be compressed
* **BestSpeed** - data will be compressed with respect to the speed
* **Default** - data will be compressed with satisfying time/compression ratio
* **BestCompression** - data will be compressed with respect to the compression

### Compression mode ###
Fourth (optional) parameter of the ZipArchiveEntry::SetCompressionStream method. Default value is **Deferred**.

* **Deferred** - the data is compressed when the ZIP archive is about to be saved. In other words, the stream instance of input data to be stored must exist when the ZIP archive will be saved. The advantage of deferred compression mode is that the compressed data *needs not* to be loaded into the memory, because they are *streamed* into the final output stream.
* **Immediate** - the data is compressed immediately into the memory buffer. It is not recommended to use this method for large files. The advantage of immediate mode is that the input stream can be destroyed (i.e. by scope) even before the ZIP archive is saved.

### Password ###
You may set the password with the ZipArchiveEntry::SetPassword method. Entry is not password protected by default.

* The structure of ZIP archive allows you to protect some specific file entries in the ZIP archive and some not
* If you want the whole ZIP archive to be password protected, you must set the password on each entry you add
* If you want to add the password protected file entry using the **immediate** mode, it is needed to set password before the SetCompressionStream call
* If you want to add the password protected file entry using the **deferred** mode, it is needed to set password before saving the archive

### Data descriptor ###
You may force the use of the data descriptor with ZipArchiveEntry::UseDataDescriptor method.

*In short:* usage of data descriptor when encrypting the entry leads to the faster compression, because the input stream is read only once, but there may be compatibility issues with very old applications.

*Explanation:* Data descriptor is small chunk of information written after the compressed data. It is most useful when storing a password protected entries. When the password protected entry does not use it, the CRC32 value of the data is required before encryption of the file data begins (it is used for checking of the correct password when decrypting), which means the stream must be read in its entirety to compute the actual CRC32 before proceeding and then be read again to actually store data.

If the data descriptor is used, the "last write time" attribute of the entry is used for checking of the correct password, therefore the stream content will be processed only once and the CRC32 is written after the compressed data.



*Note1: when you use **CompressionMethod::Stored**, the **CompressionLevel::Stored** is set automatically and vice versa.*

*Note2: data descriptor is not set by default*