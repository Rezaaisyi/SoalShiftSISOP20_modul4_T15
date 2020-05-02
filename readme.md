# SoalShiftSISOP20_modul4_T15

### Kelompok T15 :
* Anggada Putra N <05311840000025>
* Muhammad Reza Aisyi <05311840000036>

===========================================================================
```
#define FUSE_USE_VERSION 28
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <errno.h>
#include <sys/time.h>

static const char * rootDir = "/home/it/Documents";

static int xmp_getattr(const char *path, struct stat * stbuf);
static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi);
static int xmp_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi);
static int xmp_open(const char *path, struct fuse_file_info *fi);
static int xmp_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi);
static int xmp_create(const char *path, mode_t mode, struct fuse_file_info *fi);
static int xmp_truncate(const char *path, off_t size);
static struct fuse_operations xmp_oper = {
    .getattr = xmp_getattr,
    .readdir = xmp_readdir,
    .read = xmp_read,
    .open = xmp_open,
    .write = xmp_write,
    .create = xmp_create,
    .truncate = xmp_truncate,
};


int main(int argc, char *argv[]){
    umask(0);//untuk memberi hak akses pada file yang baru dibuat
    return fuse_main(argc, argv, &xmp_oper, NULL);
}


static int xmp_getattr(const char *path, struct stat * stbuf){
    int res;
    char fpath[1000];
//cek apakah direktori kita berada sudah sesuai dengan path atau belum, kalau belum file pathnya ditambahkan (rootDir+path filenya)
    if(strcmp(path, "/") == 0){
        path = rootDir;
        sprintf(fpath, "%s", path);
    } else{
        sprintf(fpath, "%s%s", rootDir, path);
    }

    res = lstat(fpath, stbuf);//untuk mendapatkan lokasi file pathnya
    
    if(res == -1){
        return -errno;
    }

    return 0;
}

static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi){
    int res;
    char fpath[1000];

    if(strcmp(path, "/") == 0){
        path = rootDir;
        sprintf(fpath, "%s", path);
    } else{
        sprintf(fpath, "%s%s", rootDir, path);
    }

    res = 0;

    DIR *dp;
    struct dirent *de;
    (void) offset;
    (void) fi;

    dp = opendir(fpath);//membuka direktori sesuai dengan filepathnya
    //jika tidak ada yang dibuka maka error (-errno ini return error pada file terakhir yang diproses)
    if(dp == NULL){
        return -errno;
    }
    //selama direktorinya bisa dibaca berdasarkan directory pathnya (tidak null) maka direktorinya akan dimasukkan kedalam struktur direktori, dan diteruskan sebagai buf
    while((de = readdir(dp)) != NULL){
        struct stat st;
        memset(&st, 0, sizeof(st));

        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;

        res = (filler(buf, de->d_name, &st, 0));//untuk melakukan entry struktur direktorinya

        if(res != 0){
            break;
        }
    }

    closedir(dp);//menutup direktori yang dibuka tadi

    return 0;
}

static int xmp_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi){
    int res, fd;
    char fpath[1000];

    if(strcmp(path, "/") == 0){
        path = rootDir;
        sprintf(fpath, "%s", path);
    } else{
        sprintf(fpath, "%s%s", rootDir, path);
    }

    res = 0;
    fd = 0;
    (void) fi;

    fd = open(fpath, O_RDONLY); //membuka file atau direktory tapi hanya untuk dibaca (read only)

    if(fd == -1){
        return -errno;
    }

    res = pread(fd, buf, size, offset); //membaca file deskriptornya berdasarkan file path diatas

    if(res == -1){
        res = - errno;
    }

    close(fd);//menutup filenya

    return res;
}

static int xmp_open(const char *path, struct fuse_file_info *fi){
    int res;
    char fpath[1000];

    if(strcmp(path, "/") == 0){
        sprintf(fpath, "%s", rootDir);
    } else{
        sprintf(fpath, "%s%s", rootDir, path);
    }

    res = open(fpath, fi->flags); //membuka file path dan menetapkan file infonya 
    //kalau error dia return error terakhirnya (menginfokan kondisi errornya dimana)
    if(res == -1){
        return -errno;
    }

    fi->fh = res;
    return 0;
}

static int xmp_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi){
    char fpath[1000];
    char fpath2[1000];
    char fname[1000];
    char fext[1000];
    char fbackup[1000];

    if(strcmp(path, "/") == 0){
        sprintf(fpath, "%s", rootDir);
    } else{
        sprintf(fpath, "%s%s", rootDir, path);
    }
    
    (void) fi;

    sprintf(fbackup, "%s/backup", rootDir); //mengisi variabel fbackup dengan isi rootDir +/backup (contoh: /home/it/Documents/backup)
    int backupfd = open(fbackup, O_RDONLY);//membuka filebackupnya
    
    if(backupfd == -1){ //jika ketika dibuka returnya error (-1) maka dia backup karena belum ada file backupnya maka untuk membukanya failed atau error 
        mkdir(fbackup, S_IRWXU | S_IRGRP | S_IXGRP | S_IROTH | S_IXOTH);//buat direktori dengan permission
    } else{
        close(backupfd);//tutup file backupnya
    }

    memset(fname, 0, sizeof(fname));
    memset(fext, 0, sizeof(fext));
    getNamaFile(fname, fext, fpath);

    time_t t;
    char ftime[1000];
    struct tm* tm_info;
    t = time(NULL);
    tm_info = localtime(&t);
    strftime(ftime, 1000, "%H:%M:%S_%d-%m-%Y", tm_info);//untuk mengambil lokal timenya saat itu yang jadi file timenya nanti

    if(strcmp(fext, "") == 0){
        sprintf(fpath2, "%s/backup/%s.%s", rootDir, fname, ftime);//jika tidak ada ekstensi langsung ditambah rootDir+/backup/+filename.+file time
    } else{
        sprintf(fpath2, "%s/backup/%s.%s.%s", rootDir, fname, ftime, fext);//lainnya (ada ekstensi) maka rootDir+/backup/+filename.+filetime.+ekstensinya
    }

    int fd1, fd2, res;
    
    if(fi == NULL){
        fd1 = open(fpath, O_WRONLY);//buka file deskriptor1 untuk ditulis (write only)
    } else{
        fd1 = fi->fh;
    }

    fd2 = open(fpath2, O_WRONLY | O_CREAT);//buka file deskriptor2 untuk ditulis dan dibuat jika tidak ada

    res = pwrite(fd1, buf, size, offset);//untuk mengisi (menulis) file deskriptor1
    pwrite(fd2, buf, size, offset);//untuk mengisi file deskriptor2
    chmod(fpath2, 0);

    if(fi == NULL){//jika tidak ada lagi file deskriptor1 maka ditutup
        close(fd1);
    }
    
    close(fd2);//menutup file deskriptor2

    if(res == -1){//jika file deskriptor1 tidak dapat diisi maka return error
        res = -errno;
    }
    
    return res;
}

static int xmp_create(const char *path, mode_t mode, struct fuse_file_info *fi){
    int res;
    char fpath[1000];

    if(strcmp(path, "/") == 0){
        sprintf(fpath, "%s", rootDir);
    } else{
        sprintf(fpath, "%s%s", rootDir, path);
    }

    res = open(fpath, fi->flags, mode);//membuka file path

    if(res == -1){
        return -errno;
    }

    fi->fh = res;
    return 0;  
}

static int xmp_truncate(const char *path, off_t size){
    int res;
    char fpath[1000];

    if(strcmp(path, "/") == 0){
        sprintf(fpath, "%s", rootDir);
    } else{
        sprintf(fpath, "%s%s", rootDir, path);
    }

    res = truncate(fpath, size);//untuk mengosongkan isi filepath

    if(res == -1){
        return -errno;
    }

    return res;
}
```

