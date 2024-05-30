#include <iostream>
#include <vector>
#include <map>
#include <assert.h>
#include <string.h>
#include <math.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>

using namespace std;
#define DISK_SIZE 512

// Function to convert decimal to binary char
char decToBinary(int n){
    return static_cast<char>(n);
}
int binaryToDec(char c) {
    return static_cast<int>(c);
}

// #define SYS_CALL
// ============================================================================
class fsInode {
    int fileSize;               // Size of the file in bytes
    int num_of_block_in_use;    // Number of blocks currently in use

    bool isDeleted;
    // Direct block pointers
    int directBlock1;
    int directBlock2;
    int directBlock3;

    int singleInDirect;     // Pointer for a single-level indirect block
    int doubleInDirect;     // Pointer for a double-level indirect block
    int block_size;         // Size of a data block in bytes

    int numOfSinglePtrAllocated;
    int numOfDoublePtrAllocated;

    int* numOfDoublePtrAllocatedAt;

public:
    // Constructor: Initialize the inode with given block size
    fsInode(int _block_size) {
        fileSize = 0;
        num_of_block_in_use = 0;
        block_size = _block_size;
        directBlock1 = -1;
        directBlock2 = -1;
        directBlock3 = -1;
        singleInDirect = -1;
        doubleInDirect = -1;
        isDeleted = false;
        numOfSinglePtrAllocated = 0;
        numOfDoublePtrAllocated = 0;
        numOfDoublePtrAllocatedAt = new int[block_size];
        for (int i = 0; i < block_size; ++i) {
            numOfDoublePtrAllocatedAt[i] = 0;
        }
    }

    int getNumOfDoublePtrAllocatedAt(int i) const {
        return numOfDoublePtrAllocatedAt[i];
    }

    void setNumOfDoublePtrAllocatedAt(int i, int num) {
        fsInode::numOfDoublePtrAllocatedAt[i] += num;
    }

    int getNumOfSinglePtrAllocated() const {
        return numOfSinglePtrAllocated;
    }

    void setNumOfSinglePtrAllocated(int num) {
        fsInode::numOfSinglePtrAllocated += num;
    }

    int getNumOfDoublePtrAllocated() const {
        return numOfDoublePtrAllocated;
    }

    void setNumOfDoublePtrAllocated(int num) {
        fsInode::numOfDoublePtrAllocated += num;
    }

    //getters and setters
    int getFileSize() const {
        return fileSize;
    }

    void setFileSize(int file_size) {
        fsInode::fileSize += file_size;
    }

    int getBlockInUse() const {
        return num_of_block_in_use;
    }

    void setBlockInUse(int blockInUse) {
        num_of_block_in_use = blockInUse;
    }

    int getDirectBlock1() const {
        return directBlock1;
    }

    void setDirectBlock1(int direct_block1) {
        fsInode::directBlock1 = direct_block1;
    }

    int getDirectBlock2() const {
        return directBlock2;
    }

    void setDirectBlock2(int direct_block2) {
        fsInode::directBlock2 = direct_block2;
    }

    int getDirectBlock3() const {
        return directBlock3;
    }

    void setDirectBlock3(int direct_block3) {
        fsInode::directBlock3 = direct_block3;
    }

    int getSingleInDirect() const {
        return singleInDirect;
    }

    void setSingleInDirect(int single_in_direct) {
        fsInode::singleInDirect = single_in_direct;
    }

    int getDoubleInDirect() const {
        return doubleInDirect;
    }

    void setDoubleInDirect(int double_in_direct) {
        fsInode::doubleInDirect = double_in_direct;
    }

    int getBlockSize() const {
        return block_size;
    }

    void setBlockSize(int blockSize) {
        if(blockSize == 1){
            return;
        }
        block_size = blockSize;
    }

    bool getIsDeleted() const {
        return isDeleted;
    }
    void setIsDeleted(bool isDeleted) {
        fsInode::isDeleted = isDeleted;
    }

    //destructor
    ~fsInode() {
        delete[] numOfDoublePtrAllocatedAt;
    }

};


// ============================================================================
class FileDescriptor {
    pair<string, fsInode*> file;  // A pair containing the file name and its associated fsInode
    bool inUse;                   // Flag indicating whether the file descriptor is in use

public:
    // Constructor: Initialize the file descriptor with the file name and associated fsInode
    FileDescriptor(string FileName, fsInode* fsi) {
        file.first = FileName;
        file.second = fsi;
        inUse = true;
    }

    // Get the file name associated with the file descriptor
    string getFileName() {
        return file.first;
    }
    //set FileName
    void setFileName(string newName) {
        file.first = newName;
    }

    // Get the fsInode associated with the file descriptor
    fsInode* getInode() {
        return file.second;
    }

    // Get the file size associated with the FileDescriptor's fsInode
    int GetFileSize() {
        if (file.second) {
            return file.second->getFileSize();
        }
        return -1; // Return -1 if no associated fsInode
    }

    // Check if the file descriptor is currently in use
    bool isInUse() {
        return inUse;
    }

    // Set the usage status of the file descriptor
    void setInUse(bool _inUse) {
        inUse = _inUse;
    }
};

#define DISK_SIM_FILE "DISK_SIM_FILE.txt"

// ============================================================================
class fsDisk {
    FILE *sim_disk_fd;
    bool is_formated;
    int blockSize;

    // BitVector - "bit" (int) vector, indicate which block in the disk is free
    //              or not.  (i.e. if BitVector[0] == 1 , means that the
    //             first block is occupied.
    int BitVectorSize;
    int *BitVector;

    int lastBlockOffset;
//    int numOfSinglePtrAllocated;
//    int numOfDoublePtrAllocated;//todo

    // Unix directories are lists of association structures,
    // each of which contains one filename and one inode number.
    map<string, fsInode*>  MainDir;

    // OpenFileDescriptors --  when you open a file,
    // the operating system creates an entry to represent that file
    // This entry number is the file descriptor.
    vector< FileDescriptor > OpenFileDescriptors;

public:
    // Constructor: Initialize the simulated disk
    fsDisk() {
        sim_disk_fd = fopen(DISK_SIM_FILE, "w+");
        assert(sim_disk_fd);

        // Initialize disk blocks with null bytes
        for (int i = 0; i < DISK_SIZE; i++) {
            int ret_val = fseek(sim_disk_fd, i, SEEK_SET);
            ret_val = fwrite("\0", 1, 1, sim_disk_fd);
            assert(ret_val == 1);
        }
        fflush(sim_disk_fd);
    }

    char* ReadFromFile(int fd, char *stringToRead, int len) {
        //determine the string
        for (int i = 0; stringToRead[i] != '\0'; ++i) {
            stringToRead[i] = '\0';
        }
        // Check if the file descriptor is within valid range
        if (fd < 0 || fd >= static_cast<int>(OpenFileDescriptors.size())) {
            cout << "Error: Invalid file descriptor." << endl;
            return stringToRead; // Return an error code
        }
        // Check if the file descriptor is actually in use
        if (!OpenFileDescriptors[fd].isInUse()) {
            cout << "Error: File descriptor is not in use." << endl;
            return stringToRead; // Return an error code
        }

        fsInode *inode = OpenFileDescriptors[fd].getInode();

        // Calculate the amount of data to read from the block
        int dataToRead = min(len, inode->getFileSize());
        // Pointer to read the string from disk
        char *stringToReadTemp = stringToRead;

        //direct block 1
        for (int i = 0; i < blockSize; ++i) {
            if(dataToRead == 0){
                return stringToRead;
            }
            fseek(sim_disk_fd, inode->getDirectBlock1() * this->blockSize + i, SEEK_SET);
            fread(stringToReadTemp, sizeof(char), 1, sim_disk_fd);
            dataToRead--;
            stringToReadTemp++;
        }
        //direct block 2
        for (int i = 0; i < blockSize; ++i) {
            if(dataToRead == 0){
                return stringToRead;
            }
            fseek(sim_disk_fd, inode->getDirectBlock2() * this->blockSize + i, SEEK_SET);
            fread(stringToReadTemp, sizeof(char), 1, sim_disk_fd);
            dataToRead--;
            stringToReadTemp++;
        }
        //direct block 3
        for (int i = 0; i < blockSize; ++i) {
            if(dataToRead == 0){
                return stringToRead;
            }
            fseek(sim_disk_fd, inode->getDirectBlock3() * this->blockSize + i, SEEK_SET);
            fread(stringToReadTemp, sizeof(char), 1, sim_disk_fd);
            dataToRead--;
            stringToReadTemp++;
        }
        //single inDirect block
        for (int i = 0; i < blockSize; ++i) {
            if(dataToRead == 0){
                return stringToRead;
            }
            char pointerIndexSingle;
            fseek(sim_disk_fd, inode->getSingleInDirect() * this->blockSize + i, SEEK_SET);
            fread(&pointerIndexSingle, sizeof(char), 1, sim_disk_fd);
            for (int j = 0; j < blockSize; ++j) {
                if(dataToRead == 0){
                    return stringToRead;
                }
                fseek(sim_disk_fd, binaryToDec(pointerIndexSingle) * this->blockSize + j, SEEK_SET);
                fread(stringToReadTemp, sizeof(char), 1, sim_disk_fd);
                dataToRead--;
                stringToReadTemp++;
            }
        }
        //Double inDirect block
        for (int i = 0; i < blockSize; ++i) {//todo
            char pointerIndexDouble1,pointerIndexDouble2;
            fseek(sim_disk_fd, inode->getDoubleInDirect() * this->blockSize + i, SEEK_SET);
            fread(&pointerIndexDouble1, sizeof(char), 1, sim_disk_fd);
            for (int j = 0; j < blockSize; ++j) {
                fseek(sim_disk_fd, binaryToDec(pointerIndexDouble1) * this->blockSize + j, SEEK_SET);
                fread(&pointerIndexDouble2, sizeof(char), 1, sim_disk_fd);
                for (int k = 0; k < blockSize; ++k) {
                    if(dataToRead == 0){
                        return stringToRead;
                    }
                    fseek(sim_disk_fd, binaryToDec(pointerIndexDouble2) * this->blockSize + k, SEEK_SET);
                    fread(stringToReadTemp, sizeof(char), 1, sim_disk_fd);
                    dataToRead--;
                    stringToReadTemp++;
                }
            }
        }

        return stringToRead; // Return the string of data read
    }

    int WriteToFile(int fd, char *data, int len) {
        if (!is_formated) {
            cout << "Error: Disk is not formatted. Cannot write to file." << endl;
            return -1; // Return an error code
        }

        // Check if the file descriptor is within valid range
        if (fd < 0 || fd >= static_cast<int>(OpenFileDescriptors.size())) {
            cout << "Error: Invalid file descriptor." << endl;
            return -1; // Return an error code
        }

        // Check if the file descriptor is actually in use
        if (!OpenFileDescriptors[fd].isInUse()) {
            cout << "Error: File descriptor is not in use." << endl;
            return -1; // Return an error code
        }

        fsInode *inode = OpenFileDescriptors[fd].getInode();

        // Calculate the maximum allowed file size
        int maxFileSize = (3 + blockSize + blockSize * blockSize) * blockSize;
        if (inode->getFileSize() + len > maxFileSize) {
            //fix the len
            len = maxFileSize - inode->getFileSize() - 1;
        }

        int currIndex = 0;// Current index within the data buffer
        for (;currIndex < len;) {
            // Calculate the current block and offset
            int currentBlock = inode->getFileSize() / blockSize;
            int offset = inode->getFileSize() % blockSize;

            // Direct blocks
            //direct block 1
            if (currentBlock < 1) {
                //if the block not allocated yet
                if (inode->getDirectBlock1() == -1) {
                    //allocating the block
                    inode->setDirectBlock1(allocateBlock());
                    //could not allocate
                    if (inode->getDirectBlock1() == -1) {
                        return -1;
                    }
                    //writing to the block
                    for (int j = 0; j < blockSize; ++j) {
                        //find the location
                        fseek(sim_disk_fd, inode->getDirectBlock1() * blockSize + j, 0);
                        //writ the data
                        fputc(data[0], sim_disk_fd);
                        //increasing the size of the file
                        inode->setFileSize(1);
                        currIndex++;
                        if (*(data + 1) == '\0') {//if we finished
                            lastBlockOffset = j + 1;
                            return 0;
                        }
                        data++;
                    }
                } else {//the directBlock1 already been in use
                    //write the data
                    for (int j = offset, k = 0; j < blockSize; ++j, k++) {//righting to the block
                        //find the location
                        fseek(sim_disk_fd, inode->getDirectBlock1() * blockSize + j, 0);
                        fputc(data[0], sim_disk_fd);//right the data
                        //increasing file size
                        inode->setFileSize(1);
                        currIndex++;
                        if (*(data + 1) == '\0') {//if we finished
                            lastBlockOffset = j + 1;
                            return 0;
                        }
                        data++;
                    }
                }
                continue;
            }
            //direct block 2
            else if (currentBlock < 2) {
                //if the block not allocated yet
                if (inode->getDirectBlock2() == -1) {
                    //allocating the block
                    inode->setDirectBlock2(allocateBlock());
                    //could not allocate
                    if (inode->getDirectBlock2() == -1) {
                        return -1;
                    }
                    //writing to the block
                    for (int j = 0; j < blockSize; ++j) {
                        fseek(sim_disk_fd, inode->getDirectBlock2() * blockSize + j, 0);//find the location
                        fputc(data[0], sim_disk_fd);
                        //increasing file size
                        inode->setFileSize(1);
                        currIndex++;
                        //if we finished
                        if (*(data + 1) == '\0') {
                            lastBlockOffset = j + 1;
                            return 0;
                        }
                        data++;
                    }
                } else {//the directBlock2 already been in use
                    //writing to the block
                    for (int j = offset, k = 0; j < blockSize; ++j, k++) {
                        fseek(sim_disk_fd, inode->getDirectBlock2() * blockSize + j, 0);//find the location
                        fputc(data[0], sim_disk_fd);
                        //increasing file size
                        inode->setFileSize(1);
                        currIndex++;
                        if (*(data + 1) == '\0') {//if we finished righting
                            lastBlockOffset = j + 1;
                            return 0;
                        }
                        data++;
                    }
                }
                continue;
            }
            //direct block 3
            else if (currentBlock < 3) {
                //if the block not allocated yet
                if (inode->getDirectBlock3() == -1) {
                    //allocating the block
                    inode->setDirectBlock3(allocateBlock());
                    //could not allocate
                    if (inode->getDirectBlock3() == -1) {
                        return -1;
                    }
                    //writing to the block
                    for (int j = 0; j < blockSize; ++j) {
                        fseek(sim_disk_fd, inode->getDirectBlock3() * blockSize + j, 0);//find the location
                        fputc(data[0], sim_disk_fd);
                        //increasing file size
                        inode->setFileSize(1);
                        currIndex++;
                        if (*(data + 1) == '\0') {//if we finished
                            lastBlockOffset = j + 1;
                            return 0;
                        }
                        data++;
                    }
                } else {//the directBlock3 already been in use
                    for (int j = offset, k = 0; j < blockSize; ++j, k++) {//righting to the block
                        fseek(sim_disk_fd, inode->getDirectBlock3() * blockSize + j, 0);//find the location
                        fputc(data[0], sim_disk_fd);//right the data
                        //increasing file size
                        inode->setFileSize(1);
                        currIndex++;
                        if (*(data + 1) == '\0') {//if we finished
                            lastBlockOffset = j + 1;
                            return 0;
                        }
                        data++;
                    }
                }
                continue;
            }
            // Single-level indirect block allocation
            else if (currentBlock < 3 + blockSize) {
                //if not allocated yet
                if (inode->getSingleInDirect() == -1) {
                    // Allocate single-level indirect block of pointers
                    inode->setSingleInDirect(allocateBlock());
                    //if there is a problem allocating
                    if (inode->getSingleInDirect() == -1) {
                        return -1;
                    }
                    //finding the location of the single indirect level
                    fseek(sim_disk_fd, inode->getSingleInDirect() * blockSize, 0);
                    //allocating the data block
                    int singlePtrData = allocateBlock();
                    //writing the ptr to the file
                    fputc(decToBinary(singlePtrData), sim_disk_fd);
                    //increasing the num of ptr
                    inode->setNumOfSinglePtrAllocated(1);
                    //writing to block
                    for (int j = 0; j < blockSize; ++j) {
                        //find the location
                        fseek(sim_disk_fd, singlePtrData * blockSize + j, 0);
                        //write the data
                        fputc(data[0], sim_disk_fd);
                        //increasing file size
                        inode->setFileSize(1);
                        currIndex++;
                        if (*(data + 1) == '\0') {//if we finished
                            lastBlockOffset = j + 1;
                            return 0;
                        }
                        data++;
                    }
                    continue;
                }
                else {//if already allocated
                    //the last block still have space to write
                    if (offset > 0) {
                        char ptr;
                        //find the current ptr
                        fseek(sim_disk_fd,inode->getSingleInDirect() * blockSize + inode->getNumOfSinglePtrAllocated() - 1,0);
                        fread(&ptr, sizeof(char),1,sim_disk_fd);
                        //write the data
                        for (int k = 0; k < blockSize - offset; ++k) {
                            //find the last block with offset
                            fseek(sim_disk_fd, binaryToDec(ptr) * blockSize + offset + k, 0);
                            fputc(data[0], sim_disk_fd);
                            //increasing file size
                            inode->setFileSize(1);
                            currIndex++;
                            if (*(data + 1) == '\0') {//if we finished writing
                                lastBlockOffset += k;
                                return 0;
                            }
                            data++;
                        }
                        lastBlockOffset = 0;
                        continue;
                    }
                    //if offset == 0
                    else {
                        //allocating new block
                        int newSingleDataBlock = allocateBlock();
                        //couldn't allocate
                        if (newSingleDataBlock == -1) {
                            return -1;
                        }
                        //find the location on the file
                        fseek(sim_disk_fd, inode->getSingleInDirect() * blockSize + inode->getNumOfSinglePtrAllocated(), 0);
                        //writing the pointer
                        fputc(decToBinary(newSingleDataBlock), sim_disk_fd);
                        //increasing the num of ptr
                        inode->setNumOfSinglePtrAllocated(1);

                        //writing to file
                        for (int k = 0; k < blockSize; ++k) {
                            //find the location of the file
                            fseek(sim_disk_fd, newSingleDataBlock * blockSize + k, 0);
                            fputc(data[0], sim_disk_fd);
                            //increasing file size
                            inode->setFileSize(1);
                            currIndex++;
                            if (*(data + 1) == '\0') {//if we finished
                                lastBlockOffset = k + 1;
                                return 0;
                            }
                            data++;
                        }
                        continue;
                    }
                }
            }
            // Handle double-level indirect block allocation
            else if (currentBlock < 3 + blockSize + blockSize * blockSize) {
                //if not allocated yet
                if (inode->getDoubleInDirect() == -1) {
                    // Allocate double-level indirect block
                    inode->setDoubleInDirect(allocateBlock());
                    //couldn't allocate
                    if (inode->getDoubleInDirect() == -1) {
                        return -1;
                    }
                    int doublePtrBlock1 = allocateBlock();
                    //couldn't allocate
                    if (doublePtrBlock1 == -1) {
                        return -1;
                    }
                    inode->setNumOfDoublePtrAllocated(1);
                    //write the pointer
                    fseek(sim_disk_fd, inode->getDoubleInDirect() * blockSize, 0);
                    fputc(decToBinary(doublePtrBlock1), sim_disk_fd);
                    //allocating the data block
                    int doublePtrBlock2 = allocateBlock();
                    if (doublePtrBlock2 == -1) {
                        return -1;
                    }
                    inode->setNumOfDoublePtrAllocatedAt(inode->getNumOfDoublePtrAllocated() -1, 1);
                    //find the 2 level block pointers
                    fseek(sim_disk_fd, doublePtrBlock1 * blockSize, 0);
                    //write the pointer
                    fputc(decToBinary(doublePtrBlock2), sim_disk_fd);
                    //writing to disk
                    for (int k = 0; k < blockSize; ++k) {
                        //find the last block
                        fseek(sim_disk_fd, doublePtrBlock2 * blockSize + k, 0);
                        fputc(data[0], sim_disk_fd);
                        //increasing file size
                        inode->setFileSize(1);
                        currIndex++;
                        if (*(data + 1) == '\0') {//if we finished
                            return 0;
                        }
                        data++;
                    }
                    continue;
                }
                else {//if allocated
                    if (offset != 0) {
                        char ptr1, ptr2;
                        //find the first ptr
                        fseek(sim_disk_fd, inode->getDoubleInDirect() * blockSize + inode->getNumOfDoublePtrAllocated() -1,0);
                        fread(&ptr1, sizeof(char),1, sim_disk_fd);
                        //find the second ptr
                        fseek(sim_disk_fd, binaryToDec(ptr1) * blockSize + inode->getNumOfDoublePtrAllocatedAt(inode->getNumOfDoublePtrAllocated() -1) -1, 0);
                        fread(&ptr2, sizeof(char),1, sim_disk_fd);
                        //write the data
                        for (int k = 0; k < blockSize - offset; ++k) {
                            //find the last block with offset
                            fseek(sim_disk_fd, binaryToDec(ptr2) * blockSize + offset + k, 0);
                            fputc(data[0], sim_disk_fd);
                            //increasing file size
                            inode->setFileSize(1);
                            currIndex++;
                            if (*(data + 1) == '\0') {//if we finished writing
                                return 0;
                            }
                            data++;
                        }
                        continue;
                    }
                    else {//if offset = 0
                        //if there is more ptr to allocate in the current double indirect
                        if(inode->getNumOfDoublePtrAllocatedAt(inode->getNumOfDoublePtrAllocated() -1) < blockSize){
                            int newDataBlock = allocateBlock();
                            if(newDataBlock == -1){
                                return -1;
                            }
                            char ptr1, ptr2;
                            //find the first ptr
                            fseek(sim_disk_fd, inode->getDoubleInDirect() * blockSize + inode->getNumOfDoublePtrAllocated() -1, 0);
                            fread(&ptr1, sizeof(char),1, sim_disk_fd);
                            //set the second ptr
                            fseek(sim_disk_fd, binaryToDec(ptr1) * blockSize + inode->getNumOfDoublePtrAllocatedAt(inode->getNumOfDoublePtrAllocated() -1), 0);
                            fputc(decToBinary(newDataBlock), sim_disk_fd);
                            inode->setNumOfDoublePtrAllocatedAt(inode->getNumOfDoublePtrAllocated() -1, 1);
                            //writing to disk
                            for (int k = 0; k < blockSize; ++k) {
                                //find the last block
                                fseek(sim_disk_fd, newDataBlock * blockSize + k, 0);
                                fputc(data[0], sim_disk_fd);
                                //increasing file size
                                inode->setFileSize(1);
                                currIndex++;
                                if (*(data + 1) == '\0') {//if we finished
                                    return 0;
                                }
                                data++;
                            }
                            continue;
                        }
                        else{//no more ptr available in the current block
                            int newPtr = allocateBlock();
                            fseek(sim_disk_fd,inode->getDoubleInDirect() * blockSize + inode->getNumOfDoublePtrAllocated(),0);
                            fputc(decToBinary(newPtr), sim_disk_fd);
                            inode->setNumOfDoublePtrAllocated(1);
                            int newDataBlock = allocateBlock();
                            fseek(sim_disk_fd, newPtr * blockSize, 0);
                            fputc(decToBinary(newDataBlock),sim_disk_fd);
                            inode->setNumOfDoublePtrAllocatedAt(inode->getNumOfDoublePtrAllocated() -1, 1);
                            //writing to disk
                            for (int k = 0; k < blockSize; ++k) {
                                //find the last block
                                fseek(sim_disk_fd, newDataBlock * blockSize + k, 0);
                                fputc(data[0], sim_disk_fd);
                                //increasing file size
                                inode->setFileSize(1);
                                currIndex++;
                                if (*(data + 1) == '\0') {//if we finished
                                    return 0;
                                }
                                data++;
                            }
                            continue;
                        }
                    }
                }
            }
        }
        // Return a success code
        return 0;
    }

    void listAll() {
        int i = 0;
        for ( auto it = begin (OpenFileDescriptors); it != end (OpenFileDescriptors); ++it) {
            cout << "index: " << i << ": FileName: " << it->getFileName() <<  " , isInUse: "
                 << it->isInUse() << " file Size: " << it->GetFileSize() << endl;
            i++;
        }
        char bufy;
        cout << "Disk content: '" ;
        for (i=0; i < DISK_SIZE ; i++) {
            int ret_val = fseek ( sim_disk_fd , i , SEEK_SET );
            ret_val = fread(  &bufy , 1 , 1, sim_disk_fd );
            cout << bufy;
        }
        cout << "'" << endl;
    }

    void fsFormat(int block_size = 4) {
        // Clear existing data structures and prepare for formatting
        for (auto& fd : OpenFileDescriptors) {
            fd.setInUse(false); // Mark all file descriptors as not in use
        }
        // Clear the Open fd
        OpenFileDescriptors.clear();
        // Clear the MainDir map
        MainDir.clear();

        // Initialize disk blocks with null bytes
        for (int i = 0; i < DISK_SIZE; i++) {
            int ret_val = fseek(sim_disk_fd, i, SEEK_SET);
            ret_val = fwrite("\0", 1, 1, sim_disk_fd);
            assert(ret_val == 1);
        }
        fflush(sim_disk_fd);

        // Update formatting status and BitVector size
        is_formated = true;
        // Calculate the number of blocks needed
        BitVectorSize = (DISK_SIZE + block_size - 1) / block_size;
        BitVector = new int[BitVectorSize];
        // Initialize BitVector with zeros
        memset(BitVector, 0, BitVectorSize * sizeof(int));
        //initialize blockSize
        this->blockSize = block_size;

        cout << "FORMAT DISK: number of blocks: " << endl;
    }

    int CreateFile(string fileName) {
        if (!is_formated) {
            cout << "Error: Disk is not formatted. Cannot create a file." << endl;
            return -1; // Return an error code
        }

        // Get the block size from the formatted disk
        int block_size = this->blockSize;

        // Create a new fsInode and add it to MainDir
        auto* newInode = new fsInode(block_size); // Create a new inode with the specified block size

        // Check if the file already exists in MainDir
        if (MainDir.find(fileName) != MainDir.end()) {
            if(MainDir.find(fileName)->second->getIsDeleted()){
                MainDir.find(fileName)->second = newInode;
                MainDir.find(fileName)->second->setIsDeleted(false);
                for (int i = 0; i < OpenFileDescriptors.size(); ++i) {
                    if(OpenFileDescriptors[i].getFileName() == fileName){
                        OpenFileDescriptors[i].setInUse(true);
                    }
                }
                return (int)(OpenFileDescriptors.size() - 1); // Return the newly assigned file descriptor
            } else {
                cout << "Error: File " << fileName << " already exists." << endl;
                return -1; // Return an error code
            }
        }

        //add ti mainDir
        MainDir[fileName] = newInode;
        // Create a new FileDescriptor and add it to OpenFileDescriptors
        FileDescriptor newFileDescriptor(fileName, newInode);
        OpenFileDescriptors.push_back(newFileDescriptor);

        return (int)(OpenFileDescriptors.size() - 1); // Return the newly assigned file descriptor
    }

    int OpenFile(string FileName) {
        if (!is_formated) {
            cout << "Error: Disk is not formatted. Cannot open a file." << endl;
            return -1; // Return an error code
        }

        // Check if the file exists in MainDir
        auto it = MainDir.find(FileName);
        if (it == MainDir.end()) {
            cout << "Error: File " << FileName << " does not exist." << endl;
            return -1; // Return an error code
        }

        // Check if the file is already open
        for (size_t i = 0; i < OpenFileDescriptors.size(); i++) {
            if (OpenFileDescriptors[i].getFileName() == FileName) {
                if(OpenFileDescriptors[i].isInUse()){
                    cout<<"ERR"<< endl;
                } else {
                    OpenFileDescriptors[i].setInUse(true);
                }
                return (int)i; // Return the existing file descriptor
            }
        }
        cout << "File " << FileName << " opened successfully with File Descriptor #: " << OpenFileDescriptors.size() - 1 << endl;
        return (int)OpenFileDescriptors.size() - 1; // Return the newly assigned file descriptor
    }

    string CloseFile(int fd) {
        // Check if the disk is formatted before attempting to close a file
        if (!is_formated) {
            return "Error: Disk is not formatted. Cannot close file.";
        }

        // Check if the file descriptor is within valid range
        if (fd < 0 || fd >= static_cast<int>(OpenFileDescriptors.size())) {
            return "Error: Invalid file descriptor.";
        }

        // Check if the file descriptor is actually in use
        if (!OpenFileDescriptors[fd].isInUse()) {
            return "Error: File descriptor is not in use.";
        }

        // Mark the file descriptor as not in use
        OpenFileDescriptors[fd].setInUse(false);

        // Get the filename associated with the file descriptor
        string fileName = OpenFileDescriptors[fd].getFileName();

        // Return a success message with the closed filename
        return "File " + fileName + " closed successfully.";
    }

    int allocateBlock(){
        for (int i = 0; i < BitVectorSize; i++) {
            if (BitVector[i] == 0) {
                BitVector[i] = 1;
                return i;
            }
        }
        return -1;
    }

    int DelFile(string FileName) {
        //check if formatted
        if (!is_formated) {
            cout << "Error: Disk is not formatted. Cannot delete file." << endl;
            return -1; // Return an error code
        }

        //check if the file exist
        auto it = MainDir.find(FileName);
        if (it == MainDir.end()) {
            cout << "Error: File " << FileName << " does not exist." << endl;
            return -1; // Return an error code
        }

        // Check if the file is open and set not inUse if it is
        for (auto & OpenFileDescriptor : OpenFileDescriptors) {
            if(OpenFileDescriptor.getFileName() == FileName) {
                if(OpenFileDescriptor.isInUse()) {
                    cout << "Error: File " << FileName << " is currently open. Close it before deleting." << endl;
                    return -1; // Return an error code
                } else{
                    OpenFileDescriptor.setInUse(false);
                }
            }
        }

        //check if the file exist
        auto it2 = MainDir.find(FileName);

        fsInode* inode = it2->second;
        //free all file the memory
        char b;
        //direct block 1
        for (int i = 0; i < blockSize; ++i) {
            if(inode->getDirectBlock1() != -1) {
                BitVector[inode->getDirectBlock1()] = 0;
            }
        }
        //direct block 2
        for (int i = 0; i < blockSize; ++i) {
            if(inode->getDirectBlock2() != -1) {
                BitVector[inode->getDirectBlock2()] = 0;
            }
        }
        //direct block 3
        for (int i = 0; i < blockSize; ++i) {
            if(inode->getDirectBlock3() != -1) {
                BitVector[inode->getDirectBlock3()] = 0;
            }
        }
        //single inDirect block
        if(inode->getSingleInDirect() != -1) {
            for (int i = 0; i < blockSize; ++i) {
                char singleDataPtr;
                fseek(sim_disk_fd, inode->getSingleInDirect() * this->blockSize + i, SEEK_SET);
                fread(&singleDataPtr, sizeof(char), 1, sim_disk_fd);
                if(BitVector[binaryToDec(singleDataPtr)] != 0) {
                    BitVector[binaryToDec(singleDataPtr)] = 0;
                }
            }
            BitVector[inode->getSingleInDirect()] = 0;
        }
        //Double inDirect block
        if(inode->getDoubleInDirect() != -1) {
            for (int i = 0; i < blockSize; ++i) {
                char pointerIndexDouble1, pointerIndexDouble2;
                fseek(sim_disk_fd, inode->getDoubleInDirect() * this->blockSize + i, SEEK_SET);
                fread(&pointerIndexDouble1, sizeof(char), 1, sim_disk_fd);
                for (int j = 0; j < blockSize; ++j) {
                    fseek(sim_disk_fd, binaryToDec(pointerIndexDouble1) * this->blockSize + j, SEEK_SET);
                    fread(&pointerIndexDouble2, sizeof(char), 1, sim_disk_fd);
                    if(BitVector[binaryToDec(pointerIndexDouble2)] != 0) {
                        BitVector[binaryToDec(pointerIndexDouble2)] = 0;
                    }
                }
                if( BitVector[binaryToDec(pointerIndexDouble1)] != 0) {
                    BitVector[binaryToDec(pointerIndexDouble1)] = 0;
                }
            }
            BitVector[inode->getDoubleInDirect()] = 0;
        }

        it->second->setIsDeleted(true);

        cout << "File " << FileName << " deleted successfully." << endl;
        return 0; // Return success code
    }

    int GetFileSize(int fd) {
        if (!is_formated) {
            cout << "Error: Disk is not formatted. Cannot get file size." << endl;
            return -1; // Return an error code
        }

        if (fd < 0 || fd >= static_cast<int>(OpenFileDescriptors.size())) {
            cout << "Error: Invalid file descriptor." << endl;
            return -1; // Return an error code
        }

        if (!OpenFileDescriptors[fd].isInUse()) {
            cout << "Error: File descriptor is not in use." << endl;
            return -1; // Return an error code
        }

        fsInode* inode = OpenFileDescriptors[fd].getInode();
        if (!inode) {
            cout << "Error: Inode associated with the file descriptor is invalid." << endl;
            return -1; // Return an error code
        }

        int fileSize = inode->getFileSize();
        cout << "File size: " << fileSize << " bytes." << endl;
        return fileSize;
    }

    // Copy a file
    int CopyFile(string srcFileName, string destFileName) {
        //check if formatted
        if (!is_formated) {
            cout << "Error: Disk is not formatted. Cannot copy files." << endl;
            return -1; // Return an error code
        }

        //check if the file exist
        auto srcIt = MainDir.find(srcFileName);
        if (srcIt == MainDir.end()) {
            cout << "Error: Source file " << srcFileName << " does not exist." << endl;
            return -1; // Return an error code
        }

        //check if the new file name exist
        if (MainDir.find(destFileName) != MainDir.end()) {
            cout << "Error: Destination file " << destFileName << " already exists." << endl;
            return -1; // Return an error code
        }

        int block_size = srcIt->second->getBlockSize();
        fsInode* srcInode = srcIt->second;
        fsInode* destInode = new fsInode(block_size); // Create a new fsInode for the destination file
        //add to mainDir
        MainDir[destFileName] = destInode;

        // Create a new FileDescriptor and add it to OpenFileDescriptors
        FileDescriptor newFileDescriptor(destFileName, destInode);
        OpenFileDescriptors.push_back(newFileDescriptor);

        //find the fd of the srcFile and the fd of the srcFile
        int fdDest, fdSrc;
        for (fdDest = 0;  OpenFileDescriptors[fdDest].getFileName() != destFileName ; fdDest++);
        for (fdSrc = 0; OpenFileDescriptors[fdSrc].getFileName() != srcFileName ; ++fdSrc);

        // Perform the actual copying logic using the fsInode's readData and writeData functions
        const int bufferSize = srcIt->second->getFileSize();
        char buffer[bufferSize];
        int bytesRead = 0;
        int totalBytesCopied = 0;

        ReadFromFile(fdSrc, buffer, bufferSize);
        WriteToFile(fdDest, buffer, bufferSize);

        MainDir[destFileName] = destInode;

        cout << "File " << srcFileName << " copied to " << destFileName << " successfully." << endl;
        return 0; // Return success code
    }

    // Rename a file
    int RenameFile(string oldFileName, string newFileName) {
        //check if formatted
        if (!is_formated) {
            cout << "Error: Disk is not formatted. Cannot rename files." << endl;
            return -1; // Return an error code
        }

        //find the file
        auto oldIt = MainDir.find(oldFileName);
        if (oldIt == MainDir.end()) {
            cout << "Error: File " << oldFileName << " does not exist." << endl;
            return -1; // Return an error code
        }

        //check if the new name already exist
        if (MainDir.find(newFileName) != MainDir.end()) {
            cout << "Error: File " << newFileName << " already exists." << endl;
            return -1; // Return an error code
        }

        fsInode* inode = oldIt->second;

        // Update the MainDir entry with the new file name
        MainDir[newFileName] = inode;
        MainDir.erase(oldIt);

        //change the name in the fd
        for (auto & OpenFileDescriptor : OpenFileDescriptors) {
            if(OpenFileDescriptor.getFileName() == oldFileName){
                OpenFileDescriptor.setFileName(newFileName);
            }
        }
        cout << "File " << oldFileName << " renamed to " << newFileName << " successfully." << endl;
        return 0; // Return success code
    }

    // Destructor
    ~fsDisk() {
        // Close the file associated with the simulated disk
        if (sim_disk_fd != nullptr) {
            fclose(sim_disk_fd);
        }

        // Deallocate the BitVector array
        delete[] BitVector;
    }
};


int main() {
    int blockSize;
    int direct_entries;
    string fileName;
    string fileName2;
    char str_to_write[DISK_SIZE];
    char str_to_read[DISK_SIZE];
    int size_to_read;
    int _fd;

    fsDisk *fs = new fsDisk();
    int cmd_;
    while(1) {
        cin >> cmd_;
        switch (cmd_)
        {
            case 0:   // exit
                delete fs;
                exit(0);
                break;

            case 1:  // list-file
                fs->listAll();
                break;

            case 2:    // format
                cin >> blockSize;
                fs->fsFormat(blockSize);
                break;

            case 3:    // creat-file
                cin >> fileName;
                _fd = fs->CreateFile(fileName);
                cout << "CreateFile: " << fileName << " with File Descriptor #: " << _fd << endl;
                break;

            case 4:  // open-file
                cin >> fileName;
                _fd = fs->OpenFile(fileName);
                cout << "OpenFile: " << fileName << " with File Descriptor #: " << _fd << endl;
                break;

            case 5:  // close-file
                cin >> _fd;
                fileName = fs->CloseFile(_fd);
                cout << "CloseFile: " << fileName << " with File Descriptor #: " << _fd << endl;
                break;

            case 6:   // write-file
                cin >> _fd;
                cin >> str_to_write;
                fs->WriteToFile( _fd , str_to_write , strlen(str_to_write) );
                break;

            case 7:    // read-file
                cin >> _fd;
                cin >> size_to_read ;
                fs->ReadFromFile( _fd , str_to_read , size_to_read );
                cout << "ReadFromFile: " << str_to_read << endl;
                break;

            case 8:   // delete file
                cin >> fileName;
                _fd = fs->DelFile(fileName);
                cout << "DeletedFile: " << fileName << " with File Descriptor #: " << _fd << endl;
                break;

            case 9:   // copy file
                cin >> fileName;
                cin >> fileName2;
                fs->CopyFile(fileName, fileName2);
                break;

            case 10:  // rename file
                cin >> fileName;
                cin >> fileName2;
                fs->RenameFile(fileName, fileName2);
                break;

            default:
                break;
        }
    }

}
