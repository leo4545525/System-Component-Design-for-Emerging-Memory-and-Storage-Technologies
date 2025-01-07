<h1>Part1.</h1>

Setting Up FUSE environment
```clike
sudo apt-get install fuse libfuse-dev
```
![image](https://hackmd.io/_uploads/rJcDQs4mJx.png)


輸入`fusermount --version`
FUSE安裝正確
![image](https://hackmd.io/_uploads/SyXcXjV7Jg.png)

<h1>Part2.</h1>
`/usr/src/linux-5.15.137`底下創建`fuse_memfs`目錄並進入
```clike
mkdir fuse_memfs  
cd    fuse_memfs
```

建立`memfs.c`
:::spoiler memfs.c
```c=
#define FUSE_USE_VERSION 26
#include <fuse.h>
#include <string.h>
#include <errno.h>
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <time.h>

#define MAX_FILES 1024        // 定義最大檔案數量
#define MAX_NAME_LEN 255      // 定義檔案名稱的最大長度

// 定義記憶體檔案系統中的檔案和目錄結構
struct memfs_file {
    char name[MAX_NAME_LEN];         // 檔案或目錄的名稱
    int is_directory;                // 是否為目錄（1 表示是目錄，0 表示檔案）
    mode_t mode;                     // 權限和模式位
    nlink_t nlink;                   // 硬連結數量
    size_t size;                     // 檔案大小（位元組）
    char *data;                      // 檔案內容的指標
    struct memfs_file *parent;       // 指向父目錄的指標
    struct memfs_file **children;    // 指向子檔案或目錄的指標陣列
    int child_count;                 // 子檔案或目錄的數量
    struct timespec atime;           // 最後訪問時間
    struct timespec mtime;           // 最後修改時間
};

static struct memfs_file *files[MAX_FILES];  // 全域檔案和目錄的指標陣列
static int file_count = 0;                   // 目前的檔案和目錄數量

// **在這裡添加 memfs_truncate 的函式原型宣告**
static int memfs_truncate(const char *path, off_t size);

// 根據給定的路徑查找對應的檔案或目錄
static struct memfs_file* find_file(const char *path) {
    if (strcmp(path, "/") == 0) {
        return files[0]; // 如果路徑是根目錄，返回根目錄的指標
    }
    char *path_copy = strdup(path); // 複製路徑字串，因為 strtok_r 會修改字串
    if (!path_copy)
        return NULL;
    char *token = NULL;
    char *rest = path_copy;
    struct memfs_file *current = files[0]; // 從根目錄開始
    while ((token = strtok_r(rest, "/", &rest))) { // 使用 '/' 分割路徑
        int found = 0;
        for (int i = 0; i < current->child_count; i++) {
            if (strcmp(current->children[i]->name, token) == 0) {
                current = current->children[i]; // 找到匹配的子節點，繼續深入
                found = 1;
                break;
            }
        }
        if (!found) {
            free(path_copy);
            return NULL; // 如果未找到，返回 NULL
        }
    }
    free(path_copy);
    return current; // 返回找到的檔案或目錄的指標
}

// 實現 getattr 操作，用於獲取檔案或目錄的屬性
static int memfs_getattr(const char *path, struct stat *stbuf) {
    memset(stbuf, 0, sizeof(struct stat)); // 清空 stat 結構
    struct memfs_file *file = find_file(path); // 查找路徑對應的檔案或目錄
    if (!file)
        return -ENOENT; // 如果未找到，返回錯誤碼
    stbuf->st_mode = file->mode;         // 設定檔案模式（權限和類型）
    stbuf->st_nlink = file->nlink;       // 設定硬連結數量
    stbuf->st_size = file->size;         // 設定檔案大小
    stbuf->st_atime = file->atime.tv_sec; // 設定最後訪問時間
    stbuf->st_mtime = file->mtime.tv_sec; // 設定最後修改時間
    stbuf->st_ctime = file->mtime.tv_sec; // 設定最後狀態改變時間
    return 0; // 成功返回 0
}

// 實現 readdir 操作，用於讀取目錄內容
static int memfs_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi) {
    (void) offset; // 未使用的參數
    (void) fi;     // 未使用的參數
    struct memfs_file *dir = find_file(path); // 查找目錄
    if (!dir || !dir->is_directory)
        return -ENOENT; // 如果未找到或不是目錄，返回錯誤

    // 添加當前目錄和父目錄的條目
    filler(buf, ".", NULL, 0);
    filler(buf, "..", NULL, 0);
    // 添加子檔案和子目錄的條目
    for (int i = 0; i < dir->child_count; i++) {
        filler(buf, dir->children[i]->name, NULL, 0);
    }
    return 0; // 成功返回 0
}

// 實現 mkdir 操作，用於創建目錄
static int memfs_mkdir(const char *path, mode_t mode) {
    char *parent_path = strdup(path); // 複製路徑字串
    if (!parent_path)
        return -ENOMEM;
    char *name = strrchr(parent_path, '/'); // 找到最後一個 '/'
    if (!name) {
        free(parent_path);
        return -EINVAL; // 無效的路徑
    }
    *name = '\0'; // 分割出父目錄路徑
    name++;
    struct memfs_file *parent = find_file(*parent_path ? parent_path : "/"); // 查找父目錄
    if (!parent || !parent->is_directory) {
        free(parent_path);
        return -ENOENT; // 父目錄不存在或不是目錄
    }
    struct memfs_file *dir = malloc(sizeof(struct memfs_file)); // 分配新的目錄結構
    if (!dir) {
        free(parent_path);
        return -ENOMEM;
    }
    memset(dir, 0, sizeof(struct memfs_file));
    strncpy(dir->name, name, MAX_NAME_LEN); // 設定目錄名稱
    dir->is_directory = 1;                  // 標記為目錄
    dir->mode = S_IFDIR | mode;             // 設定目錄的模式和權限
    dir->nlink = 2;                         // 目錄的硬連結數量（默認為 2）
    dir->parent = parent;                   // 設定父目錄
    dir->children = NULL;                   // 初始化子節點指標
    dir->child_count = 0;                   // 初始化子節點數量
    clock_gettime(CLOCK_REALTIME, &dir->atime); // 設定時間戳
    dir->mtime = dir->atime;

    // 將新目錄添加到父目錄的子節點列表
    parent->children = realloc(parent->children, sizeof(struct memfs_file*) * (parent->child_count + 1));
    if (!parent->children) {
        free(dir);
        free(parent_path);
        return -ENOMEM;
    }
    parent->children[parent->child_count++] = dir;
    parent->nlink++; // 更新父目錄的硬連結數量

    files[file_count++] = dir; // 將新目錄添加到全域檔案列表
    free(parent_path);
    return 0; // 成功返回 0
}

// 實現 rmdir 操作，用於刪除目錄
static int memfs_rmdir(const char *path) {
    struct memfs_file *dir = find_file(path); // 查找目錄
    if (!dir || !dir->is_directory)
        return -ENOENT; // 目錄不存在或不是目錄
    if (dir->child_count > 0)
        return -ENOTEMPTY; // 目錄不為空，無法刪除
    struct memfs_file *parent = dir->parent;
    if (!parent)
        return -EINVAL;

    // 從父目錄的子節點列表中移除該目錄
    for (int i = 0; i < parent->child_count; i++) {
        if (parent->children[i] == dir) {
            memmove(&parent->children[i], &parent->children[i + 1], sizeof(struct memfs_file*) * (parent->child_count - i - 1));
            parent->child_count--;
            break;
        }
    }
    parent->nlink--; // 更新父目錄的硬連結數量

    // 從全域檔案列表中移除該目錄
    for (int i = 0; i < file_count; i++) {
        if (files[i] == dir) {
            memmove(&files[i], &files[i + 1], sizeof(struct memfs_file*) * (file_count - i - 1));
            file_count--;
            break;
        }
    }
    free(dir->children); // 釋放子節點指標
    free(dir);           // 釋放目錄結構
    return 0; // 成功返回 0
}

// 實現 create 操作，用於創建檔案
static int memfs_create(const char *path, mode_t mode, struct fuse_file_info *fi) {
    (void) fi; // 未使用的參數
    struct memfs_file *existing_file = find_file(path);
    if (existing_file) {
        // 如果檔案已存在，截斷其內容
        return memfs_truncate(path, 0);
    }

    char *parent_path = strdup(path); // 複製路徑字串
    if (!parent_path)
        return -ENOMEM;
    char *name = strrchr(parent_path, '/'); // 找到最後一個 '/'
    if (!name) {
        free(parent_path);
        return -EINVAL;
    }
    *name = '\0'; // 分割出父目錄路徑
    name++;
    struct memfs_file *parent = find_file(*parent_path ? parent_path : "/"); // 查找父目錄
    if (!parent || !parent->is_directory) {
        free(parent_path);
        return -ENOENT;
    }
    struct memfs_file *file = malloc(sizeof(struct memfs_file)); // 分配新的檔案結構
    if (!file) {
        free(parent_path);
        return -ENOMEM;
    }
    memset(file, 0, sizeof(struct memfs_file));
    strncpy(file->name, name, MAX_NAME_LEN); // 設定檔案名稱
    file->is_directory = 0;                  // 標記為檔案
    file->mode = S_IFREG | mode;             // 設定檔案的模式和權限
    file->nlink = 1;                         // 檔案的硬連結數量
    file->parent = parent;                   // 設定父目錄
    file->data = NULL;                       // 初始化檔案內容指標
    file->size = 0;                          // 初始化檔案大小
    clock_gettime(CLOCK_REALTIME, &file->atime); // 設定時間戳
    file->mtime = file->atime;

    // 將新檔案添加到父目錄的子節點列表
    parent->children = realloc(parent->children, sizeof(struct memfs_file*) * (parent->child_count + 1));
    if (!parent->children) {
        free(file);
        free(parent_path);
        return -ENOMEM;
    }
    parent->children[parent->child_count++] = file;

    files[file_count++] = file; // 將新檔案添加到全域檔案列表
    free(parent_path);
    return 0; // 成功返回 0
}

// 實現 unlink 操作，用於刪除檔案
static int memfs_unlink(const char *path) {
    struct memfs_file *file = find_file(path); // 查找檔案
    if (!file || file->is_directory)
        return -ENOENT;
    struct memfs_file *parent = file->parent;
    if (!parent)
        return -EINVAL;

    // 從父目錄的子節點列表中移除該檔案
    for (int i = 0; i < parent->child_count; i++) {
        if (parent->children[i] == file) {
            memmove(&parent->children[i], &parent->children[i + 1], sizeof(struct memfs_file*) * (parent->child_count - i - 1));
            parent->child_count--;
            break;
        }
    }

    // 從全域檔案列表中移除該檔案
    for (int i = 0; i < file_count; i++) {
        if (files[i] == file) {
            memmove(&files[i], &files[i + 1], sizeof(struct memfs_file*) * (file_count - i - 1));
            file_count--;
            break;
        }
    }
    free(file->data); // 釋放檔案內容
    free(file);       // 釋放檔案結構
    return 0; // 成功返回 0
}

// 實現 open 操作，用於打開檔案
static int memfs_open(const char *path, struct fuse_file_info *fi) {
    (void) fi; // 未使用的參數
    struct memfs_file *file = find_file(path); // 查找檔案
    if (!file || file->is_directory)
        return -ENOENT;
    return 0; // 成功返回 0
}

// 實現 read 操作，用於讀取檔案內容
static int memfs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    (void) fi; // 未使用的參數
    struct memfs_file *file = find_file(path); // 查找檔案
    if (!file || file->is_directory)
        return -ENOENT;
    if (offset >= (off_t)file->size)
        return 0; // 如果偏移量超過檔案大小，返回 0
    if (offset + size > file->size)
        size = file->size - offset; // 調整讀取大小，避免越界
    memcpy(buf, file->data + offset, size); // 複製檔案內容到緩衝區

    // 更新訪問時間
    clock_gettime(CLOCK_REALTIME, &file->atime);

    return size; // 返回實際讀取的位元組數
}

// 實現 write 操作，用於寫入檔案內容
static int memfs_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    (void) fi; // 未使用的參數
    struct memfs_file *file = find_file(path); // 查找檔案
    if (!file || file->is_directory)
        return -ENOENT;
    size_t new_size = offset + size; // 計算新的檔案大小
    if (new_size > file->size) {
        // 如果需要擴展檔案大小
        char *new_data = realloc(file->data, new_size);
        if (!new_data)
            return -ENOMEM;
        memset(new_data + file->size, 0, new_size - file->size); // 將新增加的部分初始化為 0
        file->data = new_data;
    }
    memcpy(file->data + offset, buf, size); // 將資料寫入檔案
    file->size = new_size; // 更新檔案大小

    // 更新修改時間和訪問時間
    clock_gettime(CLOCK_REALTIME, &file->mtime);
    file->atime = file->mtime;

    return size; // 返回實際寫入的位元組數
}

// **memfs_truncate 函數定義**
// 實現 truncate 操作，用於截斷或擴展檔案大小
static int memfs_truncate(const char *path, off_t size)
{
    struct memfs_file *file = find_file(path); // 查找檔案
    if (!file || file->is_directory)
        return -ENOENT;

    if (size > file->size) {
        // 擴展檔案大小
        char *new_data = realloc(file->data, size);
        if (!new_data)
            return -ENOMEM;
        memset(new_data + file->size, 0, size - file->size); // 初始化新增加的部分
        file->data = new_data;
    } else if (size < file->size) {
        // 截斷檔案大小
        char *new_data = realloc(file->data, size);
        if (!new_data && size > 0)
            return -ENOMEM;
        file->data = new_data;
    }
    file->size = size; // 更新檔案大小

    // 更新修改時間和訪問時間
    clock_gettime(CLOCK_REALTIME, &file->mtime);
    file->atime = file->mtime;

    return 0; // 成功返回 0
}

// 實現 utimens 操作，用於更新檔案或目錄的時間戳
static int memfs_utimens(const char *path, const struct timespec tv[2]) {
    struct memfs_file *file = find_file(path); // 查找檔案或目錄
    if (!file)
        return -ENOENT;

    if (tv) {
        file->atime = tv[0]; // 設定訪問時間
        file->mtime = tv[1]; // 設定修改時間
    } else {
        // 如果 tv 為 NULL，使用當前時間
        clock_gettime(CLOCK_REALTIME, &file->atime);
        file->mtime = file->atime;
    }

    return 0; // 成功返回 0
}

// 初始化根目錄
static void init_root_dir() {
    struct memfs_file *root = malloc(sizeof(struct memfs_file)); // 分配根目錄結構
    if (!root)
        exit(1); // 無法分配記憶體，退出程式
    memset(root, 0, sizeof(struct memfs_file));
    strcpy(root->name, "/");          // 設定根目錄名稱
    root->is_directory = 1;           // 標記為目錄
    root->nlink = 2;                  // 根目錄的硬連結數量
    root->mode = S_IFDIR | 0755;      // 設定根目錄的模式和權限
    root->parent = NULL;              // 根目錄沒有父目錄
    root->children = NULL;            // 初始化子節點指標
    root->child_count = 0;            // 初始化子節點數量
    clock_gettime(CLOCK_REALTIME, &root->atime); // 設定時間戳
    root->mtime = root->atime;
    files[file_count++] = root;       // 將根目錄添加到全域檔案列表
}

// 定義 FUSE 操作的函數指標結構
static struct fuse_operations memfs_oper = {
    .getattr  = memfs_getattr,    // 獲取檔案屬性
    .readdir  = memfs_readdir,    // 讀取目錄
    .mkdir    = memfs_mkdir,      // 創建目錄
    .rmdir    = memfs_rmdir,      // 刪除目錄
    .create   = memfs_create,     // 創建檔案
    .unlink   = memfs_unlink,     // 刪除檔案
    .open     = memfs_open,       // 打開檔案
    .read     = memfs_read,       // 讀取檔案
    .write    = memfs_write,      // 寫入檔案
    .truncate = memfs_truncate,   // 截斷檔案
    .utimens  = memfs_utimens,    // 更新時間戳
};

// 主函數，初始化文件系統並開始運行
int main(int argc, char *argv[]) {
    init_root_dir(); // 初始化根目錄
    return fuse_main(argc, argv, &memfs_oper, NULL); // 啟動 FUSE 文件系統
}

```
:::



---


**編譯程式碼**： 確保已經進入到包含 memfs.c 的目錄後，執行以下編譯命令
```clike
gcc -Wall memfs.c `pkg-config fuse --cflags --libs` -o memfs
```

`-Wall`: 打開所有的警告訊息，方便你檢查可能存在的問題。
`pkg-config fuse --cflags --libs`: 提供 FUSE 所需的編譯和鏈結標誌。
`-o memfs`: 指定輸出的可執行檔案名為 memfs。

![image](https://hackmd.io/_uploads/r1U9tiNmyx.png)
如果編譯成功，會在當前目錄下生成名為 `memfs` 的可執行檔案，圖中綠色為可執行檔。

---
**創建掛載點**
在```fuse_memfs```目錄底下再創建一個```mountpoint```目錄，這個資料夾將用作 FUSE 檔案系統的掛載點。

為什麼掛載點需要在這個目錄中創建？
便於管理：創建掛載點在與 `memfs` 可執行檔案相同的目錄，方便進行掛載和管理這些檔案。

掛載檔案系統：這個資料夾 (mountpoint) 是 FUSE 掛載檔案系統後的入口，所有在這個資料夾內的操作都會被路由 FUSE 程式中。

創建 `mountpoint` 資料夾之後，輸入`./memfs mountpoint`，讓 `memfs` 掛載到 `mountpoint` 空目錄

也可以加入`-o nonempty` ，這樣就可以掛載到非空的掛載點目錄。`./memfs mountpoint -o nonempty`

`fusermount -u -z /usr/src/linux-5.15.137/fuse_memfs/mountpoint` 強制卸載掛載指令

`gcc -o memfs memfs.c `pkg-config fuse --cflags --libs` -lssl -lcrypto`


⚫ create files
```clike
touch mountpoint/newfile.txt
```

![image](https://hackmd.io/_uploads/B18uS3EXyl.png)
在掛載的檔案系統中建立一個名為 `newfile.txt`的空檔案


⚫ write files
```clike
echo "Hello world!" > mountpoint/newfile.txt
```
![image](https://hackmd.io/_uploads/SkSclT4XJl.png)


![image](https://hackmd.io/_uploads/SyNtxT471l.png)

確認了open.write.close files的正確性

⚫ read files
```clike
cat mountpoint/newfile.txt
```
![image](https://hackmd.io/_uploads/SJBP-T4m1l.png)



⚫ create  directories.

 創建名為`newdir`的目錄
![image](https://hackmd.io/_uploads/BJzRNa47yx.png)

⚫ remove  directories.

`rmdir`用來刪除**空目錄**
![image](https://hackmd.io/_uploads/r1KNH6N7ye.png)

若為**非空目錄**則不能刪除
![image](https://hackmd.io/_uploads/rJMMITE7Jg.png)


⚫ List directory contents.
可以看到在`mountpoint`目錄底下新創的`newdir`目錄皆可以正確執行`rmdir touch echo cat ls`這幾種操作
![image](https://hackmd.io/_uploads/S1LkyJrQkx.png)

---

<h1>Part3.</h1>

```clike
// AES-256 加密密鑰和初始向量（IV）
// 注意：在實際應用中，請實現安全的密鑰管理機制
unsigned char *key = (unsigned char *)"01234567890123456789012345678901"; // 32 字節密鑰
unsigned char *iv = (unsigned char *)"0123456789012345"; // 16 字節
```
加密和解密都是使用相同的密鑰（key）和初始向量（IV）。為對稱加密演算法 AES-256-CBC，在對稱加密中，加密和解密操作使用的是同一個密鑰。在實際應用中應實現安全的密鑰管理機制，在Part4中實作。

記得修改編譯指令為
```gcc -o memfs memfs.c `pkg-config fuse --cflags --libs` -lssl -lcrypto```
:::spoiler memfs.c修改為
```c=
#define FUSE_USE_VERSION 26
#include <fuse.h>
#include <string.h>
#include <errno.h>
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <time.h>

// 包含 OpenSSL 頭文件
#include <openssl/conf.h>
#include <openssl/evp.h>
#include <openssl/err.h>

#define MAX_FILES 1024        // 定義最大檔案數量
#define MAX_NAME_LEN 255      // 定義檔案名稱的最大長度

// 定義記憶體檔案系統中的檔案和目錄結構
struct memfs_file {
    char name[MAX_NAME_LEN];         // 檔案或目錄的名稱
    int is_directory;                // 是否為目錄（1 表示是目錄，0 表示檔案）
    mode_t mode;                     // 權限和模式位
    nlink_t nlink;                   // 硬連結數量
    size_t size;                     // 加密後的資料大小
    size_t orig_size;                // 原始（未加密）資料大小
    char *data;                      // 檔案內容的指標（加密後）
    struct memfs_file *parent;       // 指向父目錄的指標
    struct memfs_file **children;    // 指向子檔案或目錄的指標陣列
    int child_count;                 // 子檔案或目錄的數量
    struct timespec atime;           // 最後訪問時間
    struct timespec mtime;           // 最後修改時間
};

static struct memfs_file *files[MAX_FILES];  // 全域檔案和目錄的指標陣列
static int file_count = 0;                   // 目前的檔案和目錄數量

// AES-256 加密密鑰和初始向量（IV）
// 注意：在實際應用中，請實現安全的密鑰管理機制
unsigned char *key = (unsigned char *)"01234567890123456789012345678901"; // 32 字節密鑰
unsigned char *iv = (unsigned char *)"0123456789012345"; // 16 字節 IV

// 初始化 OpenSSL 庫
void init_openssl() {
    ERR_load_crypto_strings();
    OpenSSL_add_all_algorithms();
    OPENSSL_config(NULL);
}

// 清理 OpenSSL 庫
void cleanup_openssl() {
    EVP_cleanup();
    ERR_free_strings();
}

// 加密函數
int encrypt(unsigned char *plaintext, int plaintext_len, unsigned char *key,
            unsigned char *iv, unsigned char **ciphertext) {
    EVP_CIPHER_CTX *ctx;
    int len;
    int ciphertext_len;

    *ciphertext = malloc(plaintext_len + EVP_MAX_BLOCK_LENGTH);
    if (!*ciphertext)
        return -1;

    // 建立並初始化上下文
    if (!(ctx = EVP_CIPHER_CTX_new()))
        return -1;

    // 初始化加密操作，使用 AES-256-CBC 模式
    if (1 != EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, key, iv))
        return -1;

    // 提供要加密的資料，並獲取加密後的輸出
    if (1 != EVP_EncryptUpdate(ctx, *ciphertext, &len, plaintext, plaintext_len))
        return -1;
    ciphertext_len = len;

    // 完成加密操作
    if (1 != EVP_EncryptFinal_ex(ctx, *ciphertext + len, &len))
        return -1;
    ciphertext_len += len;

    // 清理
    EVP_CIPHER_CTX_free(ctx);

    return ciphertext_len;
}

// 解密函數
int decrypt(unsigned char *ciphertext, int ciphertext_len, unsigned char *key,
            unsigned char *iv, unsigned char **plaintext) {
    EVP_CIPHER_CTX *ctx;
    int len;
    int plaintext_len;

    *plaintext = malloc(ciphertext_len);
    if (!*plaintext)
        return -1;

    // 建立並初始化上下文
    if (!(ctx = EVP_CIPHER_CTX_new()))
        return -1;

    // 初始化解密操作，使用 AES-256-CBC 模式
    if (1 != EVP_DecryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, key, iv))
        return -1;

    // 提供要解密的資料，並獲取解密後的輸出
    if (1 != EVP_DecryptUpdate(ctx, *plaintext, &len, ciphertext, ciphertext_len))
        return -1;
    plaintext_len = len;

    // 完成解密操作
    if (1 != EVP_DecryptFinal_ex(ctx, *plaintext + len, &len))
        return -1;
    plaintext_len += len;

    // 清理
    EVP_CIPHER_CTX_free(ctx);

    return plaintext_len;
}

// **在這裡添加 memfs_truncate 的函式原型宣告**
static int memfs_truncate(const char *path, off_t size);

static struct memfs_file* find_file(const char *path) {
    if (strcmp(path, "/") == 0) {
        return files[0]; // 根目錄
    }
    char *path_copy = strdup(path);
    if (!path_copy)
        return NULL;
    char *token = NULL;
    char *rest = path_copy;
    struct memfs_file *current = files[0]; // 從根目錄開始
    while ((token = strtok_r(rest, "/", &rest))) {
        int found = 0;
        for (int i = 0; i < current->child_count; i++) {
            if (strcmp(current->children[i]->name, token) == 0) {
                current = current->children[i];
                found = 1;
                break;
            }
        }
        if (!found) {
            free(path_copy);
            return NULL; // 未找到
        }
    }
    free(path_copy);
    return current;
}

// 實現 getattr 操作，用於獲取檔案或目錄的屬性
static int memfs_getattr(const char *path, struct stat *stbuf) {
    memset(stbuf, 0, sizeof(struct stat));
    struct memfs_file *file = find_file(path);
    if (!file)
        return -ENOENT;
    stbuf->st_mode = file->mode;
    stbuf->st_nlink = file->nlink;
    stbuf->st_size = file->orig_size; // 返回原始資料大小
    stbuf->st_atime = file->atime.tv_sec;
    stbuf->st_mtime = file->mtime.tv_sec;
    stbuf->st_ctime = file->mtime.tv_sec;
    return 0;
}

// 實現 readdir 操作，用於讀取目錄內容
static int memfs_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi) {
    (void) offset;
    (void) fi;
    struct memfs_file *dir = find_file(path);
    if (!dir || !dir->is_directory)
        return -ENOENT;

    filler(buf, ".", NULL, 0);
    filler(buf, "..", NULL, 0);
    for (int i = 0; i < dir->child_count; i++) {
        filler(buf, dir->children[i]->name, NULL, 0);
    }
    return 0;
}

// 實現 mkdir 操作，用於創建目錄
static int memfs_mkdir(const char *path, mode_t mode) {
    char *parent_path = strdup(path);
    if (!parent_path)
        return -ENOMEM;
    char *name = strrchr(parent_path, '/');
    if (!name) {
        free(parent_path);
        return -EINVAL;
    }
    *name = '\0';
    name++;
    struct memfs_file *parent = find_file(*parent_path ? parent_path : "/");
    if (!parent || !parent->is_directory) {
        free(parent_path);
        return -ENOENT;
    }
    struct memfs_file *dir = malloc(sizeof(struct memfs_file));
    if (!dir) {
        free(parent_path);
        return -ENOMEM;
    }
    memset(dir, 0, sizeof(struct memfs_file));
    strncpy(dir->name, name, MAX_NAME_LEN);
    dir->is_directory = 1;
    dir->mode = S_IFDIR | mode;
    dir->nlink = 2;
    dir->parent = parent;
    dir->children = NULL;
    dir->child_count = 0;
    clock_gettime(CLOCK_REALTIME, &dir->atime);
    dir->mtime = dir->atime;

    parent->children = realloc(parent->children, sizeof(struct memfs_file*) * (parent->child_count + 1));
    if (!parent->children) {
        free(dir);
        free(parent_path);
        return -ENOMEM;
    }
    parent->children[parent->child_count++] = dir;
    parent->nlink++; // 更新父目錄的 nlink

    files[file_count++] = dir;
    free(parent_path);
    return 0;
}

// 實現 rmdir 操作，用於刪除目錄
static int memfs_rmdir(const char *path) {
    struct memfs_file *dir = find_file(path);
    if (!dir || !dir->is_directory)
        return -ENOENT;
    if (dir->child_count > 0)
        return -ENOTEMPTY;
    struct memfs_file *parent = dir->parent;
    if (!parent)
        return -EINVAL;

    // 從父目錄的 children 中移除
    for (int i = 0; i < parent->child_count; i++) {
        if (parent->children[i] == dir) {
            memmove(&parent->children[i], &parent->children[i + 1], sizeof(struct memfs_file*) * (parent->child_count - i - 1));
            parent->child_count--;
            break;
        }
    }
    parent->nlink--; // 更新父目錄的 nlink

    // 從全域 files 中移除
    for (int i = 0; i < file_count; i++) {
        if (files[i] == dir) {
            memmove(&files[i], &files[i + 1], sizeof(struct memfs_file*) * (file_count - i - 1));
            file_count--;
            break;
        }
    }
    free(dir->children);
    free(dir);
    return 0;
}

// 實現 create 操作，用於創建檔案
static int memfs_create(const char *path, mode_t mode, struct fuse_file_info *fi) {
    (void) fi;
    struct memfs_file *existing_file = find_file(path);
    if (existing_file) {
        // 如果檔案已存在，截斷其內容
        return memfs_truncate(path, 0);
    }

    char *parent_path = strdup(path);
    if (!parent_path)
        return -ENOMEM;
    char *name = strrchr(parent_path, '/');
    if (!name) {
        free(parent_path);
        return -EINVAL;
    }
    *name = '\0';
    name++;
    struct memfs_file *parent = find_file(*parent_path ? parent_path : "/");
    if (!parent || !parent->is_directory) {
        free(parent_path);
        return -ENOENT;
    }
    struct memfs_file *file = malloc(sizeof(struct memfs_file));
    if (!file) {
        free(parent_path);
        return -ENOMEM;
    }
    memset(file, 0, sizeof(struct memfs_file));
    strncpy(file->name, name, MAX_NAME_LEN);
    file->is_directory = 0;
    file->mode = S_IFREG | mode;
    file->nlink = 1;
    file->parent = parent;
    file->data = NULL;
    file->size = 0;
    file->orig_size = 0;
    clock_gettime(CLOCK_REALTIME, &file->atime);
    file->mtime = file->atime;

    parent->children = realloc(parent->children, sizeof(struct memfs_file*) * (parent->child_count + 1));
    if (!parent->children) {
        free(file);
        free(parent_path);
        return -ENOMEM;
    }
    parent->children[parent->child_count++] = file;

    files[file_count++] = file;
    free(parent_path);
    return 0;
}

// 實現 unlink 操作，用於刪除檔案
static int memfs_unlink(const char *path) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;
    struct memfs_file *parent = file->parent;
    if (!parent)
        return -EINVAL;

    // 從父目錄的 children 中移除
    for (int i = 0; i < parent->child_count; i++) {
        if (parent->children[i] == file) {
            memmove(&parent->children[i], &parent->children[i + 1], sizeof(struct memfs_file*) * (parent->child_count - i - 1));
            parent->child_count--;
            break;
        }
    }

    // 從全域 files 中移除
    for (int i = 0; i < file_count; i++) {
        if (files[i] == file) {
            memmove(&files[i], &files[i + 1], sizeof(struct memfs_file*) * (file_count - i - 1));
            file_count--;
            break;
        }
    }
    free(file->data);
    free(file);
    return 0;
}

// 實現 open 操作，用於打開檔案
static int memfs_open(const char *path, struct fuse_file_info *fi) {
    (void) fi;
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;
    return 0;
}

// 實現 read 操作，用於讀取檔案內容
static int memfs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    (void) fi;
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    if (offset >= (off_t)file->orig_size)
        return 0;

    if (offset + size > file->orig_size)
        size = file->orig_size - offset;

    // 打印加密的資料（以十六進制顯示）
    printf("Encrypted data in memory (hex): ");
    for (size_t i = 0; i < file->size; i++) {
        printf("%02x ", (unsigned char)file->data[i]);
    }
    printf("\n");

    // 解密資料
    unsigned char *decrypted_data = NULL;
    int decrypted_size = decrypt((unsigned char *)file->data, file->size, key, iv, &decrypted_data);
    if (decrypted_size < 0) {
        free(decrypted_data);
        return -EIO;
    }

    // 打印解密後的資料
    printf("Decrypted data: %.*s\n", decrypted_size, decrypted_data);

    memcpy(buf, decrypted_data + offset, size);
    free(decrypted_data);

    // 更新訪問時間
    clock_gettime(CLOCK_REALTIME, &file->atime);

    return size;
}

// 實現 write 操作，用於寫入檔案內容
static int memfs_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    (void) fi;
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 解密現有資料
    unsigned char *decrypted_data = NULL;
    int decrypted_size = 0;
    if (file->data && file->size > 0) {
        decrypted_size = decrypt((unsigned char *)file->data, file->size, key, iv, &decrypted_data);
        if (decrypted_size < 0) {
            free(decrypted_data);
            return -EIO;
        }
    } else {
        decrypted_data = calloc(1, offset + size);
        decrypted_size = offset + size;
    }

    // 擴展緩衝區
    if ((size_t)(offset + size) > (size_t)decrypted_size) {
        decrypted_data = realloc(decrypted_data, offset + size);
        memset(decrypted_data + decrypted_size, 0, (offset + size) - decrypted_size);
        decrypted_size = offset + size;
    }

    // 寫入新資料
    memcpy(decrypted_data + offset, buf, size);

    // 打印原始資料
    printf("Original data to write: %.*s\n", (int)(offset + size), decrypted_data);

    // 加密資料
    unsigned char *encrypted_data = NULL;
    int encrypted_size = encrypt(decrypted_data, decrypted_size, key, iv, &encrypted_data);
    if (encrypted_size < 0) {
        free(decrypted_data);
        free(encrypted_data);
        return -EIO;
    }

    // 打印加密後的資料（以十六進制顯示）
    printf("Encrypted data (hex): ");
    for (int i = 0; i < encrypted_size; i++) {
        printf("%02x ", (unsigned char)encrypted_data[i]);
    }
    printf("\n\n");

    // 更新檔案結構
    free(file->data);
    file->data = (char *)encrypted_data;
    file->size = encrypted_size;
    file->orig_size = decrypted_size;

    free(decrypted_data);

    // 更新時間戳
    clock_gettime(CLOCK_REALTIME, &file->mtime);
    file->atime = file->mtime;

    return size;
}

// **memfs_truncate 函數定義**
// 實現 truncate 操作，用於截斷或擴展檔案大小
static int memfs_truncate(const char *path, off_t size) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 解密現有資料
    unsigned char *decrypted_data = NULL;
    int decrypted_size = 0;
    if (file->data && file->size > 0) {
        decrypted_size = decrypt((unsigned char *)file->data, file->size, key, iv, &decrypted_data);
        if (decrypted_size < 0) {
            free(decrypted_data);
            return -EIO;
        }
    } else {
        decrypted_data = calloc(1, size);
        decrypted_size = size;
    }

    // 調整資料大小
    decrypted_data = realloc(decrypted_data, size);
    if (size > decrypted_size)
        memset(decrypted_data + decrypted_size, 0, size - decrypted_size);
    decrypted_size = size;

    // 加密資料
    unsigned char *encrypted_data = NULL;
    int encrypted_size = encrypt(decrypted_data, decrypted_size, key, iv, &encrypted_data);
    if (encrypted_size < 0) {
        free(decrypted_data);
        free(encrypted_data);
        return -EIO;
    }

    // 更新檔案結構
    free(file->data);
    file->data = (char *)encrypted_data;
    file->size = encrypted_size;
    file->orig_size = decrypted_size;

    free(decrypted_data);

    // 更新時間戳
    clock_gettime(CLOCK_REALTIME, &file->mtime);
    file->atime = file->mtime;

    return 0;
}

// 實現 utimens 操作，用於更新檔案或目錄的時間戳
static int memfs_utimens(const char *path, const struct timespec tv[2]) {
    struct memfs_file *file = find_file(path);
    if (!file)
        return -ENOENT;

    if (tv) {
        file->atime = tv[0];
        file->mtime = tv[1];
    } else {
        // 如果 tv 為 NULL，使用當前時間
        clock_gettime(CLOCK_REALTIME, &file->atime);
        file->mtime = file->atime;
    }

    return 0;
}

// 初始化根目錄
static void init_root_dir() {
    struct memfs_file *root = malloc(sizeof(struct memfs_file));
    if (!root)
        exit(1);
    memset(root, 0, sizeof(struct memfs_file));
    strcpy(root->name, "/");
    root->is_directory = 1;
    root->nlink = 2;
    root->mode = S_IFDIR | 0755;
    root->parent = NULL;
    root->children = NULL;
    root->child_count = 0;
    clock_gettime(CLOCK_REALTIME, &root->atime);
    root->mtime = root->atime;
    files[file_count++] = root;
}

// 定義 FUSE 操作的函數指標結構
static struct fuse_operations memfs_oper = {
    .getattr  = memfs_getattr,    // 獲取檔案屬性
    .readdir  = memfs_readdir,    // 讀取目錄
    .mkdir    = memfs_mkdir,      // 創建目錄
    .rmdir    = memfs_rmdir,      // 刪除目錄
    .create   = memfs_create,     // 創建檔案
    .unlink   = memfs_unlink,     // 刪除檔案
    .open     = memfs_open,       // 打開檔案
    .read     = memfs_read,       // 讀取檔案
    .write    = memfs_write,      // 寫入檔案
    .truncate = memfs_truncate,   // 截斷檔案
    .utimens  = memfs_utimens,    // 更新時間戳
};

// 主函數，初始化文件系統並開始運行
int main(int argc, char *argv[]) {
    init_openssl();    // 初始化 OpenSSL

    init_root_dir();   // 初始化根目錄
    int ret = fuse_main(argc, argv, &memfs_oper, NULL);

    cleanup_openssl(); // 清理 OpenSSL
    return ret;
}

```
:::
如果用`./memfs mountpoint -o nonempty`文件系統將在背景模式下運行，程式的標準輸出和標準錯誤輸出通常不會顯示在終端中。無法直接在終端中看到加密和解密的輸出

所以用`./memfs -f mountpoint -o nonempty` 時，`-f`選項，表示前台運行（foreground）。這意味著所有的輸出（printf資訊，包括加密和解密的結果）都會直接顯示在運行該命令的終端中。此終端中不會有明顯的輸出，但程式已經在等待處理文件系統請求。

接著開啟另一個終端，對掛載點中的文件系統進行操作
```clike
echo "Hello, Encrypted World!" > mountpoint/encfile.txt
cat mountpoint/encfile.txt
```
![image](https://hackmd.io/_uploads/r1_4JinmJe.png)

print出的結果驗證加密和解密的正確性

---
<h1>Part4後</h1>

每個不同的file都有不同的key
:::spoiler memfs.c改為
```c=
#define FUSE_USE_VERSION 26
#include <fuse.h>
#include <string.h>
#include <errno.h>
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <time.h>

// 包含 OpenSSL 头文件，用于加密功能
#include <openssl/conf.h>
#include <openssl/evp.h>
#include <openssl/err.h>
#include <openssl/rand.h>

#define MAX_FILES 1024        // 定义最大文件数量
#define MAX_NAME_LEN 255      // 定义文件名称的最大长度

// 定义内存文件系统中的文件和目录结构
struct memfs_file {
    char name[MAX_NAME_LEN];         // 文件或目录的名称
    int is_directory;                // 是否为目录（1 表示是目录，0 表示文件）
    mode_t mode;                     // 权限和模式位
    nlink_t nlink;                   // 硬链接数量
    size_t size;                     // 文件大小（加密后的大小）
    char *data;                      // 文件内容的指针（加密后的数据）
    struct memfs_file *parent;       // 指向父目录的指针
    struct memfs_file **children;    // 指向子文件或目录的指针数组
    int child_count;                 // 子文件或目录的数量
    struct timespec atime;           // 最后访问时间
    struct timespec mtime;           // 最后修改时间
    unsigned char iv[16];            // 初始化向量（IV），用于加密
    unsigned char key[32];           // **新增字段，文件的加密密钥（256 位）**
};

static struct memfs_file *files[MAX_FILES];  // 全局文件和目录的指针数组
static int file_count = 0;                   // 当前的文件和目录数量

// 初始化 OpenSSL 库
void init_openssl() {
    OpenSSL_add_all_algorithms();
    ERR_load_crypto_strings();
}

// 清理 OpenSSL 库
void cleanup_openssl() {
    EVP_cleanup();
    ERR_free_strings();
}

// 加密函数
int encrypt(unsigned char *plaintext, int plaintext_len, unsigned char *key,
            unsigned char *iv, unsigned char **ciphertext) {
    EVP_CIPHER_CTX *ctx; // 使用指针
    int len;
    int ciphertext_len;

    *ciphertext = malloc(plaintext_len + EVP_MAX_BLOCK_LENGTH);
    if (!*ciphertext)
        return -1;

    // 创建并初始化上下文
    if (!(ctx = EVP_CIPHER_CTX_new()))
        return -1;

    // 初始化加密操作，使用 AES-256-CBC 模式
    if (1 != EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, key, iv)) {
        EVP_CIPHER_CTX_free(ctx);
        return -1;
    }

    // 提供要加密的数据，并获取加密后的输出
    if (1 != EVP_EncryptUpdate(ctx, *ciphertext, &len, plaintext, plaintext_len)) {
        EVP_CIPHER_CTX_free(ctx);
        return -1;
    }
    ciphertext_len = len;

    // 完成加密操作
    if (1 != EVP_EncryptFinal_ex(ctx, *ciphertext + len, &len)) {
        EVP_CIPHER_CTX_free(ctx);
        return -1;
    }
    ciphertext_len += len;

    // 清理
    EVP_CIPHER_CTX_free(ctx);

    return ciphertext_len;
}

int decrypt(unsigned char *ciphertext, int ciphertext_len, unsigned char *key,
            unsigned char *iv, unsigned char **plaintext) {
    EVP_CIPHER_CTX *ctx; // 使用指针
    int len;
    int plaintext_len;

    *plaintext = malloc(ciphertext_len + EVP_MAX_BLOCK_LENGTH);
    if (!*plaintext)
        return -1;

    // 创建并初始化上下文
    if (!(ctx = EVP_CIPHER_CTX_new()))
        return -1;

    // 初始化解密操作，使用 AES-256-CBC 模式
    if (1 != EVP_DecryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, key, iv)) {
        EVP_CIPHER_CTX_free(ctx);
        return -1;
    }

    // 提供要解密的数据，并获取解密后的输出
    if (1 != EVP_DecryptUpdate(ctx, *plaintext, &len, ciphertext, ciphertext_len)) {
        EVP_CIPHER_CTX_free(ctx);
        return -1;
    }
    plaintext_len = len;

    // 完成解密操作
    if (1 != EVP_DecryptFinal_ex(ctx, *plaintext + len, &len)) {
        EVP_CIPHER_CTX_free(ctx);
        return -1;
    }
    plaintext_len += len;

    // 清理
    EVP_CIPHER_CTX_free(ctx);

    return plaintext_len;
}


// 生成随机密钥
int generate_random_key(unsigned char *key, int length) {
    if (!RAND_bytes(key, length)) {
        return -1;
    }
    return 0;
}

// 函数原型声明
static int memfs_truncate(const char *path, off_t size);
static struct memfs_file* find_file(const char *path);

// 其他函数的原型声明
static int memfs_getattr(const char *path, struct stat *stbuf);
static int memfs_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi);
static int memfs_mkdir(const char *path, mode_t mode);
static int memfs_rmdir(const char *path);
static int memfs_create(const char *path, mode_t mode, struct fuse_file_info *fi);
static int memfs_unlink(const char *path);
static int memfs_open(const char *path, struct fuse_file_info *fi);
static int memfs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi);
static int memfs_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi);
static int memfs_release(const char *path, struct fuse_file_info *fi);
static int memfs_utimens(const char *path, const struct timespec tv[2]);
static void init_root_dir();

// 实现 find_file 函数
static struct memfs_file* find_file(const char *path) {
    if (strcmp(path, "/") == 0) {
        return files[0]; // 根目录
    }
    char *path_copy = strdup(path);
    if (!path_copy)
        return NULL;
    char *token = NULL;
    char *rest = path_copy;
    struct memfs_file *current = files[0]; // 从根目录开始
    while ((token = strtok_r(rest, "/", &rest))) {
        int found = 0;
        for (int i = 0; i < current->child_count; i++) {
            if (strcmp(current->children[i]->name, token) == 0) {
                current = current->children[i];
                found = 1;
                break;
            }
        }
        if (!found) {
            free(path_copy);
            return NULL; // 未找到
        }
    }
    free(path_copy);
    return current;
}

// 实现 getattr 操作
static int memfs_getattr(const char *path, struct stat *stbuf) {
    memset(stbuf, 0, sizeof(struct stat));
    struct memfs_file *file = find_file(path);
    if (!file)
        return -ENOENT;
    stbuf->st_mode = file->mode;
    stbuf->st_nlink = file->nlink;
    stbuf->st_size = file->size; // 返回加密后的数据大小
    stbuf->st_atime = file->atime.tv_sec;
    stbuf->st_mtime = file->mtime.tv_sec;
    stbuf->st_ctime = file->mtime.tv_sec;
    return 0;
}

// 实现 readdir 操作
static int memfs_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi) {
    (void) offset;
    (void) fi;
    struct memfs_file *dir = find_file(path);
    if (!dir || !dir->is_directory)
        return -ENOENT;

    filler(buf, ".", NULL, 0);
    filler(buf, "..", NULL, 0);
    for (int i = 0; i < dir->child_count; i++) {
        filler(buf, dir->children[i]->name, NULL, 0);
    }
    return 0;
}

// 实现 mkdir 操作
static int memfs_mkdir(const char *path, mode_t mode) {
    char *parent_path = strdup(path);
    if (!parent_path)
        return -ENOMEM;
    char *name = strrchr(parent_path, '/');
    if (!name) {
        free(parent_path);
        return -EINVAL;
    }
    *name = '\0';
    name++;
    struct memfs_file *parent = find_file(*parent_path ? parent_path : "/");
    if (!parent || !parent->is_directory) {
        free(parent_path);
        return -ENOENT;
    }
    struct memfs_file *dir = malloc(sizeof(struct memfs_file));
    if (!dir) {
        free(parent_path);
        return -ENOMEM;
    }
    memset(dir, 0, sizeof(struct memfs_file));
    strncpy(dir->name, name, MAX_NAME_LEN);
    dir->is_directory = 1;
    dir->mode = S_IFDIR | mode;
    dir->nlink = 2;
    dir->parent = parent;
    dir->children = NULL;
    dir->child_count = 0;
    clock_gettime(CLOCK_REALTIME, &dir->atime);
    dir->mtime = dir->atime;

    parent->children = realloc(parent->children, sizeof(struct memfs_file*) * (parent->child_count + 1));
    if (!parent->children) {
        free(dir);
        free(parent_path);
        return -ENOMEM;
    }
    parent->children[parent->child_count++] = dir;
    parent->nlink++; // 更新父目录的 nlink

    files[file_count++] = dir;
    free(parent_path);
    return 0;
}

// 实现 rmdir 操作
static int memfs_rmdir(const char *path) {
    struct memfs_file *dir = find_file(path);
    if (!dir || !dir->is_directory)
        return -ENOENT;
    if (dir->child_count > 0)
        return -ENOTEMPTY;
    struct memfs_file *parent = dir->parent;
    if (!parent)
        return -EINVAL;

    // 从父目录的 children 中移除
    for (int i = 0; i < parent->child_count; i++) {
        if (parent->children[i] == dir) {
            memmove(&parent->children[i], &parent->children[i + 1], sizeof(struct memfs_file*) * (parent->child_count - i - 1));
            parent->child_count--;
            break;
        }
    }
    parent->nlink--; // 更新父目录的 nlink

    // 从全局 files 中移除
    for (int i = 0; i < file_count; i++) {
        if (files[i] == dir) {
            memmove(&files[i], &files[i + 1], sizeof(struct memfs_file*) * (file_count - i - 1));
            file_count--;
            break;
        }
    }
    free(dir->children);
    free(dir);
    return 0;
}

// 实现 create 操作
static int memfs_create(const char *path, mode_t mode, struct fuse_file_info *fi) {
    struct memfs_file *existing_file = find_file(path);
    if (existing_file) {
        // 如果文件已存在，截断其内容
        return memfs_truncate(path, 0);
    }

    char *parent_path = strdup(path);
    if (!parent_path)
        return -ENOMEM;
    char *name = strrchr(parent_path, '/');
    if (!name) {
        free(parent_path);
        return -EINVAL;
    }
    *name = '\0';
    name++;
    struct memfs_file *parent = find_file(*parent_path ? parent_path : "/");
    if (!parent || !parent->is_directory) {
        free(parent_path);
        return -ENOENT;
    }
    struct memfs_file *file = malloc(sizeof(struct memfs_file));
    if (!file) {
        free(parent_path);
        return -ENOMEM;
    }
    memset(file, 0, sizeof(struct memfs_file));
    strncpy(file->name, name, MAX_NAME_LEN);
    file->is_directory = 0;
    file->mode = S_IFREG | mode;
    file->nlink = 1;
    file->parent = parent;
    file->data = NULL;
    file->size = 0;
    clock_gettime(CLOCK_REALTIME, &file->atime);
    file->mtime = file->atime;

    // 生成随机的初始化向量（IV）
    if (!RAND_bytes(file->iv, 16)) {
        free(file);
        free(parent_path);
        return -EIO;
    }

    // **生成随机的文件密钥**
    if (generate_random_key(file->key, 32) != 0) {
        free(file);
        free(parent_path);
        return -EIO;
    }

    // **仅用于验证：打印文件密钥（不要在实际应用中这样做）**
    printf("Created file '%s' with key: ", path);
    for (int i = 0; i < 32; i++) {
        printf("%02x", file->key[i]);
    }
    printf("\n");

    // 将新文件添加到父目录的子节点列表
    parent->children = realloc(parent->children, sizeof(struct memfs_file*) * (parent->child_count + 1));
    if (!parent->children) {
        free(file);
        free(parent_path);
        return -ENOMEM;
    }
    parent->children[parent->child_count++] = file;

    files[file_count++] = file;
    free(parent_path);

    return 0;
}

// 实现 unlink 操作
static int memfs_unlink(const char *path) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;
    struct memfs_file *parent = file->parent;
    if (!parent)
        return -EINVAL;

    // 从父目录的 children 中移除
    for (int i = 0; i < parent->child_count; i++) {
        if (parent->children[i] == file) {
            memmove(&parent->children[i], &parent->children[i + 1], sizeof(struct memfs_file*) * (parent->child_count - i - 1));
            parent->child_count--;
            break;
        }
    }

    // 从全局 files 中移除
    for (int i = 0; i < file_count; i++) {
        if (files[i] == file) {
            memmove(&files[i], &files[i + 1], sizeof(struct memfs_file*) * (file_count - i - 1));
            file_count--;
            break;
        }
    }

    free(file->data);
    free(file);

    return 0;
}

// 实现 open 操作
static int memfs_open(const char *path, struct fuse_file_info *fi) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 不需要用户输入密钥，密钥保存在文件结构中
    return 0;
}

// 实现 read 操作
static int memfs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 解密数据
    unsigned char *plaintext = NULL;
    int plaintext_len = 0;
    if (file->data && file->size > 0) {
        plaintext_len = decrypt((unsigned char *)file->data, file->size, file->key, file->iv, &plaintext);
        if (plaintext_len < 0) {
            fprintf(stderr, "Decryption failed for file %s.\n", path);
            free(plaintext);
            return -EIO;
        }
    } else {
        // 文件为空
        return 0;
    }

    if (offset >= plaintext_len) {
        free(plaintext);
        return 0;
    }
    if (offset + size > plaintext_len)
        size = plaintext_len - offset;

    memcpy(buf, plaintext + offset, size);

    free(plaintext);

    // 更新访问时间
    clock_gettime(CLOCK_REALTIME, &file->atime);

    return size;
}
/* static int memfs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 制造一个错误的key（全零）
    unsigned char wrong_key[32];
    memset(wrong_key, 0, sizeof(wrong_key));

    // 尝试使用错误的key解密数据
    unsigned char *plaintext = NULL;
    int plaintext_len = 0;
    if (file->data && file->size > 0) {
        plaintext_len = decrypt((unsigned char *)file->data, file->size, wrong_key, file->iv, &plaintext);
        if (plaintext_len < 0) {
            fprintf(stderr, "Decryption failed for file %s with a wrong key.\n", path);
            free(plaintext);
            return -EIO; // 返回 I/O 错误
        }
    } else {
        // 文件为空
        return 0;
    }

    if (offset >= plaintext_len) {
        free(plaintext);
        return 0;
    }
    if (offset + size > plaintext_len)
        size = plaintext_len - offset;

    memcpy(buf, plaintext + offset, size);

    free(plaintext);

    // 更新访问时间
    clock_gettime(CLOCK_REALTIME, &file->atime);

    return size;
} */


// 实现 write 操作
static int memfs_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 解密现有数据
    unsigned char *plaintext = NULL;
    int plaintext_len = 0;
    if (file->data && file->size > 0) {
        plaintext_len = decrypt((unsigned char *)file->data, file->size, file->key, file->iv, &plaintext);
        if (plaintext_len < 0) {
            fprintf(stderr, "Decryption failed for file %s.\n", path);
            free(plaintext);
            return -EIO;
        }
    } else {
        plaintext = malloc(offset + size);
        plaintext_len = offset + size;
        memset(plaintext, 0, plaintext_len);
    }

    // 扩展缓冲区
    if (offset + size > plaintext_len) {
        plaintext = realloc(plaintext, offset + size);
        memset(plaintext + plaintext_len, 0, (offset + size) - plaintext_len);
        plaintext_len = offset + size;
    }

    // 写入新数据
    memcpy(plaintext + offset, buf, size);

    // 加密数据
    unsigned char *ciphertext = NULL;
    int ciphertext_len = encrypt(plaintext, plaintext_len, file->key, file->iv, &ciphertext);
    if (ciphertext_len < 0) {
        fprintf(stderr, "Encryption failed for file %s.\n", path);
        free(plaintext);
        free(ciphertext);
        return -EIO;
    }

    // 更新文件内容
    free(file->data);
    file->data = (char *)ciphertext;
    file->size = ciphertext_len;

    free(plaintext);

    // 更新修改时间和访问时间
    clock_gettime(CLOCK_REALTIME, &file->mtime);
    file->atime = file->mtime;

    return size;
}

// 实现 truncate 操作
static int memfs_truncate(const char *path, off_t size) {
    // 对于加密文件系统，需要解密数据，调整大小后再加密
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 解密现有数据
    unsigned char *plaintext = NULL;
    int plaintext_len = 0;
    if (file->data && file->size > 0) {
        plaintext_len = decrypt((unsigned char *)file->data, file->size, file->key, file->iv, &plaintext);
        if (plaintext_len < 0) {
            fprintf(stderr, "Decryption failed for file %s.\n", path);
            free(plaintext);
            return -EIO;
        }
    } else {
        plaintext = malloc(size);
        memset(plaintext, 0, size);
        plaintext_len = size;
    }

    // 调整大小
    if (size < plaintext_len) {
        plaintext_len = size;
    } else if (size > plaintext_len) {
        plaintext = realloc(plaintext, size);
        memset(plaintext + plaintext_len, 0, size - plaintext_len);
        plaintext_len = size;
    }

    // 加密数据
    unsigned char *ciphertext = NULL;
    int ciphertext_len = encrypt(plaintext, plaintext_len, file->key, file->iv, &ciphertext);
    if (ciphertext_len < 0) {
        fprintf(stderr, "Encryption failed for file %s.\n", path);
        free(plaintext);
        free(ciphertext);
        return -EIO;
    }

    // 更新文件内容
    free(file->data);
    file->data = (char *)ciphertext;
    file->size = ciphertext_len;

    free(plaintext);

    // 更新修改时间和访问时间
    clock_gettime(CLOCK_REALTIME, &file->mtime);
    file->atime = file->mtime;

    return 0;
}

// 实现 utimens 操作
static int memfs_utimens(const char *path, const struct timespec tv[2]) {
    struct memfs_file *file = find_file(path);
    if (!file)
        return -ENOENT;

    if (tv) {
        file->atime = tv[0];
        file->mtime = tv[1];
    } else {
        // 如果 tv 为 NULL，使用当前时间
        clock_gettime(CLOCK_REALTIME, &file->atime);
        file->mtime = file->atime;
    }

    return 0;
}

// 实现 release 操作（可选，不需要做任何操作）
static int memfs_release(const char *path, struct fuse_file_info *fi) {
    (void) path;
    (void) fi;
    return 0;
}

// 初始化根目录
static void init_root_dir() {
    struct memfs_file *root = malloc(sizeof(struct memfs_file));
    if (!root)
        exit(1);
    memset(root, 0, sizeof(struct memfs_file));
    strcpy(root->name, "/");
    root->is_directory = 1;
    root->nlink = 2;
    root->mode = S_IFDIR | 0755;
    root->parent = NULL;
    root->children = NULL;
    root->child_count = 0;
    clock_gettime(CLOCK_REALTIME, &root->atime);
    root->mtime = root->atime;
    files[file_count++] = root;
}

// 定义 FUSE 操作的函数指针结构
static struct fuse_operations memfs_oper = {
    .getattr  = memfs_getattr,
    .readdir  = memfs_readdir,
    .mkdir    = memfs_mkdir,
    .rmdir    = memfs_rmdir,
    .create   = memfs_create,
    .unlink   = memfs_unlink,
    .open     = memfs_open,
    .read     = memfs_read,
    .write    = memfs_write,
    .truncate = memfs_truncate,
    .utimens  = memfs_utimens,
    .release  = memfs_release,
};

// 主函数
int main(int argc, char *argv[]) {
    init_openssl(); // 初始化 OpenSSL

    init_root_dir(); // 初始化根目录

    int ret = fuse_main(argc, argv, &memfs_oper, NULL);

    cleanup_openssl(); // 清理 OpenSSL

    return ret;
}

```
:::

![image](https://hackmd.io/_uploads/Sy4NdXeVJe.png)

額外測試encryption key must be supplied to open a file正確性，
嘗試使用錯誤的key解密data
:::spoiler  memfs_read()改為
```c=
static int memfs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    struct memfs_file *file = find_file(path);
    if (!file || file->is_directory)
        return -ENOENT;

    // 制造一个错误的key（全零）
    unsigned char wrong_key[32];
    memset(wrong_key, 0, sizeof(wrong_key));

    // 尝试使用错误的key解密数据
    unsigned char *plaintext = NULL;
    int plaintext_len = 0;
    if (file->data && file->size > 0) {
        plaintext_len = decrypt((unsigned char *)file->data, file->size, wrong_key, file->iv, &plaintext);
        if (plaintext_len < 0) {
            fprintf(stderr, "Decryption failed for file %s with a wrong key.\n", path);
            free(plaintext);
            return -EIO; // 返回 I/O 错误
        }
    } else {
        // 文件为空
        return 0;
    }

    if (offset >= plaintext_len) {
        free(plaintext);
        return 0;
    }
    if (offset + size > plaintext_len)
        size = plaintext_len - offset;

    memcpy(buf, plaintext + offset, size);

    free(plaintext);

    // 更新访问时间
    clock_gettime(CLOCK_REALTIME, &file->atime);

    return size;
}
```


:::
![image](https://hackmd.io/_uploads/rk4SFXgV1g.png)




