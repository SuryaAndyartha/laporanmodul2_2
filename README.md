---

# Pipip's Load Balancer

Pipip, seorang pengembang perangkat lunak yang tengah mengerjakan proyek distribusi pesan dengan sistem load balancing, memutuskan untuk merancang sebuah sistem yang memungkinkan pesan dari client bisa disalurkan secara efisien ke beberapa worker. Dengan menggunakan komunikasi antar-proses (IPC), Pipip ingin memastikan bahwa proses pengiriman pesan berjalan mulus dan terorganisir dengan baik, melalui sistem log yang tercatat dengan rapi.

### **a. Client Mengirimkan Pesan ke Load Balancer**

Pipip ingin agar proses `client.c` dapat mengirimkan pesan ke `loadbalancer.c` menggunakan IPC dengan metode **shared memory**. Proses pengiriman pesan dilakukan dengan format input dari pengguna sebagai berikut:

```
Halo A;10
```

**Penjelasan:**

- `"Halo A"` adalah isi pesan yang akan dikirim.
- `10` adalah jumlah pesan yang ingin dikirim, dalam hal ini sebanyak 10 kali pesan yang sama.

Selain itu, setiap kali pesan dikirim, proses `client.c` harus menuliskan aktivitasnya ke dalam **`sistem.log`** dengan format:

```
Message from client: <isi pesan>
Message count: <jumlah pesan>
```

Semua pesan yang dikirimkan dari client akan diteruskan ke `loadbalancer.c` untuk diproses lebih lanjut.

### **b. Load Balancer Mendistribusikan Pesan ke Worker Secara Round-Robin**

Setelah menerima pesan dari client, tugas `loadbalancer.c` adalah mendistribusikan pesan-pesan tersebut ke beberapa **worker** menggunakan metode **round-robin**. Sebelum mendistribusikan pesan, `loadbalancer.c` terlebih dahulu mencatat informasi ke dalam **`sistem.log`** dengan format:

```
Received at lb: <isi pesan> (#message <indeks pesan>)
```

Contoh jika ada 10 pesan yang dikirimkan, maka output log yang dihasilkan adalah:

```
Received at lb: Halo A (#message 1)
Received at lb: Halo A (#message 2)
...
Received at lb: Halo A (#message 10)
```

Setelah itu, `loadbalancer.c` akan meneruskan pesan-pesan tersebut ke **n worker** secara bergiliran (round-robin), menggunakan **IPC message queue**. Berikut adalah contoh distribusi jika jumlah worker adalah 3:

- Pesan 1 → worker1
- Pesan 2 → worker2
- Pesan 3 → worker3
- Pesan 4 → worker1 (diulang dari awal)

Dan seterusnya.

Proses `worker.c` bertugas untuk mengeksekusi pesan yang diterima dan mencatat log ke dalam file yang sama, yakni **`sistem.log`**.

### **c. Worker Mencatat Pesan yang Diterima**

Setiap worker yang menerima pesan dari `loadbalancer.c` harus mencatat pesan yang diterima ke dalam **`sistem.log`** dengan format log sebagai berikut:

```
WorkerX: message received
```

### **d. Catat Total Pesan yang Diterima Setiap Worker di Akhir Eksekusi**

Setelah proses selesai (semua pesan sudah diproses), setiap worker akan mencatat jumlah total pesan yang mereka terima ke bagian akhir file **`sistem.log`**.

```
Worker 1: 3 messages
Worker 2: 4 messages
Worker 3: 3 messages
```

**Penjelasan:**
3 + 4 + 3 = 10, sesuai dengan jumlah pesan yang dikirim pada soal a

---

### Penyelesaian

### a. client.c; Code Lengkap:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>

#define MAX_STRING 512

typedef struct SharedMessage{
    char message[MAX_STRING];
    int message_counter;
} SharedMessage;

int main(){
    char input[MAX_STRING];
    char *message;
    char *message_counter_str;
    int message_counter;
    
    scanf("%[^\n]s", input);

    message = strtok(input, ";");
    message_counter_str = strtok(NULL, ";");
    message_counter = atoi(message_counter_str);

    key_t key = 1234;
    int shmid = shmget(key, sizeof(SharedMessage), IPC_CREAT | 0666);
    if(shmid == -1){
        printf("shmget gagal.\n");
        return 1;
    }

    void* shared_memory = shmat(shmid, NULL, 0);
    if(shared_memory == (void *)-1){
        printf("shmat gagal.\n");
        return 2;
    }

    SharedMessage *data = (SharedMessage*)shared_memory;
    
    strcpy(data->message, message);
    data->message_counter = message_counter;

    FILE *log_file = fopen("sistem.log", "a");
    if(log_file == NULL){
        printf("fopen gagal.\n");
        return 3;
    }
    else{
        fprintf(log_file, "Message from client: %s\n", message);
        fprintf(log_file, "Message count: %d\n", message_counter);
        fclose(log_file);
    }

    shmdt(shared_memory);

    return 0;
}
```

### Penjelasan:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
```
Kode dimulai oleh beberapa _library_ penting untuk menjalankan program sesuai yang diinginkan. `#include <stdio.h>` menyediakan akses ke fungsi `scanf`, `printf`, dan `fopen`. `#include <string.h>` menyediakan fungsi `strcpy` dan `strtok`. `#include <stdlib.h>` menyediakan fungsi `atoi`. `#include <unistd.h>` menyediakan fungsi `shmdt`. `#include <sys/ipc.h>` dan `#include <sys/shm.h>` menyediakan akses ke fungsi `shmget` dan `shmat`.

```c
#define MAX_STRING 512
```
Bagian ini mendefinisikan ukuran maksimal _array_ karakter yang akan digunakan untuk menyimpan _string_ sebagai 512 dengan `#define MAX_STRING 512`.

```c
typedef struct SharedMessage{
    char message[MAX_STRING];
    int message_counter;
} SharedMessage;
```
Di bagian ini, kode akan membuat sebuah `struct` dengan `typedef struct SharedMessage`, yang berfungsi untuk mengelompokkan dua data: `message`, yaitu _array_ karakter berukuran `MAX_STRING` yang akan menyimpan pesan dari _input_, dan `message_counter`, yaitu variabel bertipe `int` yang akan menyimpan jumlah pesan.

```c
int main(){
    char input[MAX_STRING];
    char *message;
    char *message_counter_str;
    int message_counter;
    ...
}
```
`int main(){...}` adalah fungsi utama yang menjadi titik masuk eksekusi program. Di dalamnya, program dimulai dan dieksekusi.
Setelahnya, terdapat bagian yang mendeklarasikan variabel-variabel untuk keperluan pengolahan _input_. `char input[MAX_STRING]` untuk menyimpan `input` dari pengguna. `char *message` adalah _pointer_ yang akan menunjuk isi pesan. `char *message_counter_str` adalah _pointer_ yang akan menunjuk jumlah pesan dalam bentuk _string_. `int message_counter` akan menyimpan jumlah pesan setelah dikonversi dari _string_.

```c
scanf("%[^\n]s", input);

message = strtok(input, ";");
message_counter_str = strtok(NULL, ";");
message_counter = atoi(message_counter_str);
```
Potongan kode ini akan _input_ menggunakan `scanf("%[^\n]s", input)`. `[^\n]` memastikan `scanf` membaca semua karakter sampai menemui newline `(\n)`. Setelah itu, `message = strtok(input, ";")` akan memotong _input_ berdasarkan tanda titik koma `(;)` dan menyimpan potongan pertama, yaitu pesan, ke dalam _pointer_ `message`. `message_counter_str = strtok(NULL, ";")` mengambil potongan berikutnya setelah tanda titik koma, yaitu jumlah pesan dalam bentuk _string_, dan menyimpannya ke dalam `message_counter_str`. Terakhir, `message_counter = atoi(message_counter_str)` mampu mengubah _string_ `message_counter_str` menjadi _integer_ dan menyimpannya ke dalam variabel `message_counter` menggunakan fungsi `atoi` (_ascii to integer_).

```c
key_t key = 1234;
int shmid = shmget(key, sizeof(SharedMessage), IPC_CREAT | 0666);
if(shmid == -1){
    printf("shmget gagal.\n");
    return 1;
}
```
Bagian ini akan mendeklarasikan sebuah variabel `key` bertipe `key_t` dan memberinya nilai `1234` untuk mengakses _shared memory_. Berikutnya ada fungsi `shmget` yang memiliki tiga parameter. Parameter pertama adalah `key`. Parameter kedua adalah `sizeof(SharedMessage)`, yang menentukan ukuran dari _shared memory_, dalam kasus ini ukuran `SharedMessage`. Parameter ketiga adalah `flag`, yaitu `IPC_CREAT` untuk membuat _shared memory_ jika belum ada dan `0666` untuk menentukan izin akses (_read and write_). Jika `shmget` gagal, maka `shmid` akan bernilai `-1`, sehingga program akan mencetak pesan `"shmget gagal."` dan mengembalikan nilai `1` sebagai tanda kegagalan.

```c
void* shared_memory = shmat(shmid, NULL, 0);
if(shared_memory == (void *)-1){
    printf("shmat gagal.\n");
    return 2;
}
```
`void* shared_memory` adalah sebuah _pointer_ bertipe `void*` (tipe ini mampu menyimpan berbagai tipe data) yang digunakan untuk menyimpan alamat memori dari _shared memory_. Variabel ini akan menyimpan nilai yang dikembalikan oleh fungsi `shmat` dengan tiga parameter. Parameter pertama adalah `shmid`, yaitu ID _shared memory_. Parameter kedua adalah `NULL` yang berarti sistem akan memilih alamat yang sesuai untuk pemetaan. Parameter ketiga adalah `flag`, yang diatur ke 0 untuk pemetaan standar. Jika gagal, `shmat` akan mengembalikan `(void *)-1`, sehingga program akan mencetak pesan `"shmat gagal."` dan mengembalikan `nilai 2` sebagai tanda kegagalan. (Nilai `return` yang berbeda hanya digunakan untuk memudahkan pemeriksaan kesalahan/_debugging_).

```c
SharedMessage *data = (SharedMessage*)shared_memory;
```
`SharedMessage *data = (SharedMessage*)shared_memory` adalah konversi (_casting_) _pointer_ `shared_memory` bertipe `void*` ke tipe _pointer_ `SharedMessage*`. Ini memungkinkan akses langsung ke `struct SharedMessage` yang ada di dalam _shared memory_.

```c
strcpy(data->message, message);
data->message_counter = message_counter;
```
Di bagian ini, fungsi `strcpy` digunakan untuk menyalin _string_ dari `message` ke dalam elemen `data->message` yang berada di `struct SharedMessage`. Selanjutnya, pada `data->message_counter = message_counter`, nilai dari variabel `message_counter` yang telah dihitung sebelumnya (jumlah pesan) disalin ke dalam elemen `data->message_counter`. Dengan begini, informasi pesan dan jumlahnya bisa disimpan dengan baik ke dalam _shared memory_ yang nantinya bisa diakses oleh proses lain.

```c
FILE *log_file = fopen("sistem.log", "a");
if(log_file == NULL){
    printf("fopen gagal.\n");
    return 3;
}
else{
    fprintf(log_file, "Message from client: %s\n", message);
    fprintf(log_file, "Message count: %d\n", message_counter);
    fclose(log_file);
}
```
`FILE *log_file = fopen("sistem.log", "a")` akan membuka _log file_ bernama `sistem.log` dalam mode _append_ (`"a"`) menggunakan fungsi `fopen`. Mode _append_ berarti data baru akan ditambahkan ke akhir _file_ jika _file_ tersebut sudah ada. Jika _file_ tidak ada, maka _file_ baru akan dibuat. Jika _log file_ gagal dibuka (`if(log_file == NULL)`), akan dicetak pesan `"fopen gagal."` dan program mengembalikan nilai dengan `return 3` sebagai tanda kegagalan. Jika _file_ berhasil dibuka (`else{...}`), maka program akan menulis pesan menggunakan `fprintf(log_file, "Message from client: %s\n", message)` dan jumlah pesan dengan `fprintf(log_file, "Message count: %d\n", message_counter)` ke _log file_. Setelah selesai menulis, _file_ ditutup menggunakan `fclose(log_file)` untuk memastikan tidak ada kebocoran sumber daya atau hal tidak diinginkan lainnya.

```c
shmdt(shared_memory);
```
Fungsi `shmdt` adalah fungsi yang digunakan untuk melepaskan _shared memory_ yang sebelumnya telah dipetakan. Fungsi ini memastikan bahwa proses tidak lagi mengakses _shared memory_ tersebut. 

```c
return 0;
```
Kode diakhiri oleh `return 0` yang menandakan bahwa program telah berhasil dieksekusi tanpa _error_.

### Foto Hasil Output

Sebelum:
![image alt](https://github.com/SuryaAndyartha/laporanmodul2_2/blob/main/Screenshot%202025-04-29%20at%2007.06.58.png?raw=true)
