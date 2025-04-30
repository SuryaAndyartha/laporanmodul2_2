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

### a.  Client Mengirimkan Pesan ke Load Balancer (client.c); Code Lengkap:

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

![image alt](https://github.com/SuryaAndyartha/laporanmodul2_2/blob/main/Screenshot%202025-04-29%20at%2007.06.58.png?raw=true)

### b.  Load Balancer Mendistribusikan Pesan ke Worker Secara Round-Robin (loadbalancer.c); Code Lengkap:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/msg.h>

#define MAX_STRING 512

typedef struct SharedMessage{
    char message[MAX_STRING];
    int message_counter;
} SharedMessage;

typedef struct {
    int worker_count;
    int message_count;
} Data_for_Message_Queue; 

typedef struct {
    long int worker_number;
    char message[MAX_STRING];
} Message_for_Worker;

int main()
{
    // Shared Memory untuk pesan dari client.c
    key_t key = 1234;
    int shmid1 = shmget(key, sizeof(SharedMessage), IPC_CREAT | 0666);
    if(shmid1 == -1){
        printf("shmget gagal.\n");
        return 1;
    }

    void* client_message = shmat(shmid1, NULL, 0);
    if(client_message == (void *)-1){
        printf("shmat gagal.\n");
        return 2;
    }

    SharedMessage *client_data =  (SharedMessage*)client_message;

    FILE *log_file = fopen("sistem.log", "a");
    if(log_file == NULL){
        printf("fopen gagal.\n");
        return 3;
    }

    for (int i = 1; i <= client_data->message_counter; i++)
    {
        fprintf(log_file, "Received at lb: %s (#message %d)\n", client_data->message, i);
    }

    fclose(log_file);
    shmctl(shmid1, IPC_RMID, NULL);

    int worker_count;
    scanf("%d", &worker_count);
 
    // Shared Memory untuk worker
    key = 4321;
    int shmid2 = shmget(key, sizeof(Data_for_Message_Queue), IPC_CREAT | 0666);
    if(shmid2 == -1){
        printf("shmget gagal.\n");
        return 1;
    }

    void* shared_memory_for_worker = shmat(shmid2, NULL, 0);
    if(shared_memory_for_worker == (void *)-1){
        printf("shmat gagal.\n");
        return 2;
    }

    Data_for_Message_Queue *data = (Data_for_Message_Queue*) shared_memory_for_worker;

    data->message_count = client_data->message_counter;
    data->worker_count = worker_count;

    // Message Queue untuk worker
    Message_for_Worker buff;
    int msgid;
    key = 2143;
    msgid = msgget(key, 0666 | IPC_CREAT);
    int index_worker = 1;
    for (int i = 1; i <= client_data->message_counter; i++)
    {
        if (index_worker > worker_count) index_worker -= worker_count;
        buff.worker_number = index_worker;
        strcpy(buff.message, client_data->message);
        msgsnd(msgid, &buff, sizeof(MAX_STRING), 0);
        index_worker++;
    }
    
    shmdt(shared_memory_for_worker);
    shmdt(client_message);

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
#include <sys/msg.h>
```
Kode dimulai oleh beberapa _library_ penting untuk menjalankan program sesuai yang diinginkan. `#include <stdio.h>` menyediakan fungsi `printf`, `fprintf`, dan `fopen`. `#include <string.h>` menyediakan fungsi `strcpy`. `#include <stdlib.h>` menyediakan fungsi `exit`. `#include <unistd.h>` menyediakan fungsi `shmdt`. `#include <sys/ipc.h>` menyediakan `key_t`. `#include <sys/shm.h>` menyediakan fungsi `shmget`, `shmat`, dan `shmctl`. `#include <sys/msg.h>` menyediakan fungsi `msgget` dan `msgsnd`.

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
Di bagian ini, kode akan membuat sebuah `struct` dengan `typedef struct SharedMessage`, yang berfungsi untuk mengelompokkan dua data: `message`, yaitu _array_ karakter berukuran `MAX_STRING` yang akan menyimpan pesan _input_ dari _client_, dan `message_counter`, yaitu variabel bertipe `int` yang akan menyimpan jumlah pesan.

```c
typedef struct {
    int worker_count;
    int message_count;
} Data_for_Message_Queue; 
```
Di bagian ini, kode akan membuat sebuah `struct` dengan `typedef struct Data_for_Message_Queue`, yang berfungsi untuk mengelompokkan dua data: `worker_count`, yaitu variabel bertipe `int` untuk menyimpan jumlah _worker_ yang tersedia, dan `message_count` yang merupakan variabel bertipe `int` juga untuk menyimpan total jumlah pesan dari _client_.

```c
typedef struct {
    long int worker_number;
    char message[MAX_STRING];
} Message_for_Worker;
```
Di bagian ini, kode akan membuat sebuah `struct` dengan `typedef struct Message_for_Worker`, yang berfungsi untuk mengelompokkan dua data: `worker_number`, yaitu variabel bertipe `long int` untuk menunjukkan nomor _worker_ tujuan, dan `message`, yaitu  _array_ karakter berukuran `MAX_STRING` untuk menyimpan isi pesan yang akan dikirim ke _worker_.

```c
int main()
{
    // Shared Memory untuk pesan dari client.c
    key_t key = 1234;
    int shmid1 = shmget(key, sizeof(SharedMessage), IPC_CREAT | 0666);
    if(shmid1 == -1){
        printf("shmget gagal.\n");
        return 1;
    }
    ...
}
```
`int main(){...}` adalah fungsi utama yang menjadi titik masuk eksekusi program. Di dalamnya, program dimulai dan dieksekusi.
Setelahnya, terdapat bagian yang akan mendeklarasikan sebuah variabel `key` bertipe `key_t` dan memberinya nilai `1234` untuk mengakses _shared memory_ antara _client_ dan _load balancer_. Berikutnya ada fungsi `shmget` yang memiliki tiga parameter. Parameter pertama adalah `key`. Parameter kedua adalah `sizeof(SharedMessage)`, yang menentukan ukuran dari _shared memory_, dalam kasus ini ukuran `SharedMessage`. Parameter ketiga adalah `flag`, yaitu `IPC_CREAT` untuk membuat _shared memory_ jika belum ada dan `0666` untuk menentukan izin akses (_read and write_). Jika `shmget` gagal, maka `shmid1` akan bernilai `-1`, sehingga program akan mencetak pesan `"shmget gagal."` dan mengembalikan nilai `1` sebagai tanda kegagalan.

```c
void* client_message = shmat(shmid1, NULL, 0);
if(client_message == (void *)-1){
    printf("shmat gagal.\n");
    return 2;
}
```
`void* client_message` adalah sebuah _pointer_ bertipe `void*` (tipe ini mampu menyimpan berbagai tipe data) yang digunakan untuk menyimpan alamat memori dari _shared memory_ antara _client_ dan _load balancer_. Variabel ini akan menyimpan nilai yang dikembalikan oleh fungsi `shmat` dengan tiga parameter. Parameter pertama adalah `shmid1`, yaitu ID _shared memory_. Parameter kedua adalah `NULL` yang berarti sistem akan memilih alamat yang sesuai untuk pemetaan. Parameter ketiga adalah `flag`, yang diatur ke 0 untuk pemetaan standar. Jika gagal, `shmat` akan mengembalikan `(void *)-1`, sehingga program akan mencetak pesan `"shmat gagal."` dan mengembalikan `nilai 2` sebagai tanda kegagalan. (Nilai `return` yang berbeda hanya digunakan untuk memudahkan pemeriksaan kesalahan/_debugging_).

```c
SharedMessage *client_data =  (SharedMessage*)client_message;
```
`SharedMessage *client_data = (SharedMessage*)client_message` adalah konversi (_casting_) _pointer_ `client_message` bertipe `void*` ke tipe _pointer_ `SharedMessage*`. Ini memungkinkan akses langsung ke `struct SharedMessage` yang ada di dalam _shared memory_.

```c
FILE *log_file = fopen("sistem.log", "a");
if(log_file == NULL){
    printf("fopen gagal.\n");
    return 3;
}
```
`FILE *log_file = fopen("sistem.log", "a")` akan membuka _log file_ bernama `sistem.log` dalam mode _append_ (`"a"`) menggunakan fungsi `fopen`. Mode _append_ berarti data baru akan ditambahkan ke akhir _file_ jika _file_ tersebut sudah ada. Jika _file_ tidak ada, maka _file_ baru akan dibuat. Jika _log file_ gagal dibuka (`if(log_file == NULL)`), akan dicetak pesan `"fopen gagal."` dan program mengembalikan nilai dengan `return 3` sebagai tanda kegagalan. 

```c
for (int i = 1; i <= client_data->message_counter; i++)
{
    fprintf(log_file, "Received at lb: %s (#message %d)\n", client_data->message, i);
}
```
Jika pembukaan _log file_ tidak gagal, maka akan ada bagian yang digunakan untuk mencatat `log` sebanyak jumlah `message_counter` (dari indeks pertama atau `i = 1` sampai batas `message_counter` atau `i <= client_data->message_counter`). Di dalam _loop_ yang menggunakan `for`, fungsi `fprintf` dipanggil untuk menulis ke _log file_ dengan format yang sudah disimpan oleh _pointer_ `client_data->message` dan nomor urutan pesan atau indeks ke-`i` itu sendiri.

```c
fclose(log_file);
shmctl(shmid1, IPC_RMID, NULL);
```
Setelah selesai menulis, _file_ ditutup menggunakan `fclose(log_file)` untuk memastikan tidak ada kebocoran sumber daya atau hal tidak diinginkan lainnya. Lalu akan ada baris yang memanggil `shmctl` untuk menghapus _shared memory segment_ dengan ID `shmid1`. Parameter `IPC_RMID` menunjukkan bahwa memori tersebut akan dihapus, dan paramter `NULL` berarti program tidak membutuhkan `struct` tambahan.

```c
int worker_count;
scanf("%d", &worker_count);
```
Di bagian ini, terjadi deklarasi variabel bertipe `int` dengan nama `worker_count` yang akan menyimpan berapa banyak jumlah _worker_ yang akan melakukan proses berikutnya. Fungsi `scanf` digunakan untuk meminta _user input_ secara _arbitrary_. 

```c
key = 4321;
int shmid2 = shmget(key, sizeof(Data_for_Message_Queue), IPC_CREAT | 0666);
if(shmid2 == -1){
    printf("shmget gagal.\n");
    return 1;
}
```
Bagian ini akan mendeklarasikan sebuah variabel `key` bertipe `key_t` dan memberinya nilai `4321` (hanya sebagai pembeda) untuk mengakses _shared memory_ antara _load balancer_ dan _worker_. Berikutnya ada fungsi `shmget` yang memiliki tiga parameter. Parameter pertama adalah `key`. Parameter kedua adalah `sizeof(Data_for_Message_Queue)`, yang menentukan ukuran dari _shared memory_, dalam kasus ini ukuran `Data_for_Message_Queue`. Parameter ketiga adalah `flag`, yaitu `IPC_CREAT` untuk membuat _shared memory_ jika belum ada dan `0666` untuk menentukan izin akses (_read and write_). Jika `shmget` gagal, maka `shmid2` akan bernilai `-1`, sehingga program akan mencetak pesan `"shmget gagal."` dan mengembalikan nilai `1` sebagai tanda kegagalan.

```c
void* shared_memory_for_worker = shmat(shmid2, NULL, 0);
if(shared_memory_for_worker == (void *)-1){
    printf("shmat gagal.\n");
    return 2;
}
```
`void* shared_memory_for_worker` adalah sebuah _pointer_ bertipe `void*` (tipe ini mampu menyimpan berbagai tipe data) yang digunakan untuk menyimpan alamat memori dari _shared memory_ antara _load balancer_ dan _worker_. Variabel ini akan menyimpan nilai yang dikembalikan oleh fungsi `shmat` dengan tiga parameter. Parameter pertama adalah `shmid2`, yaitu ID _shared memory_. Parameter kedua adalah `NULL` yang berarti sistem akan memilih alamat yang sesuai untuk pemetaan. Parameter ketiga adalah `flag`, yang diatur ke 0 untuk pemetaan standar. Jika gagal, `shmat` akan mengembalikan `(void *)-1`, sehingga program akan mencetak pesan `"shmat gagal."` dan mengembalikan `nilai 2` sebagai tanda kegagalan. 

```c
Data_for_Message_Queue *data = (Data_for_Message_Queue*) shared_memory_for_worker;
```
`Data_for_Message_Queue *data = (Data_for_Message_Queue*) shared_memory_for_worker` adalah konversi (_casting_) _pointer_ `shared_memory_for_worker` bertipe `void*` ke tipe _pointer_ `Data_for_Message_Queue*`. Ini memungkinkan akses langsung ke `struct Data_for_Message_Queue` yang ada di dalam _shared memory_.

```c
data->message_count = client_data->message_counter;
data->worker_count = worker_count;
```
Bagian ini menyimpan informasi penting ke dalam _shared memory_ yang akan diakses oleh proses _worker_. Nilai `message_count` dari _client_ disimpan ke `data->message_count`, dan jumlah _worker_ yang dimasukkan _user_ disimpan ke `data->worker_count`.

```c
// Message Queue untuk worker
Message_for_Worker buff;
int msgid;
key = 2143;
msgid = msgget(key, 0666 | IPC_CREAT);
int index_worker = 1;
```
Bagian ini bertujuan untuk menyiapkan _message queue_ yang akan digunakan untuk mengirim pesan ke masing-masing _worker_. Pertama, dibuat sebuah variabel `buff` bertipe `struct Message_for_Worker` sebagai _buffer_ untuk menyimpan pesan sebelum dikirim. Kemudian, `key` diatur ke `2143` (hanya sebagai pembeda). Setelah itu, fungsi `msgget(key, 0666 | IPC_CREAT)` dipanggil dan hasilnya disimpan dalam `msgid`. `msgget` memiliki dua parameter: parameter pertama adalah `key` dan parameter kedua adalah `flag` yang terdiri dari `0666` (izin _read_ and _write_) serta `IPC_CREAT` untuk membuat _message queue_ jika belum ada. Terakhir, `index_worker` diatur ke `1` sebagai penanda awal _worker_ yang akan menerima pesan pertama, yang akan digunakan dalam _round robin_ distribusi pesan.

```c
for (int i = 1; i <= client_data->message_counter; i++)
{
    ...
}
```
Bagian ini menjalankan proses pengiriman pesan ke _worker_ secara bergilir menggunakan metode _round robin_ dengan `for loop`. Perulangan dilakukan sebanyak jumlah pesan yang dikirim _client_ (`client_data->message_counter`). Di dalam _loop_, akan dilakukan:

   - ```c
     if (index_worker > worker_count) index_worker -= worker_count;
     ```
     Pemeriksaan apakah `index_worker` melebihi jumlah _worker_ (`worker_count)`. Jika iya, maka `index_worker` dikurangi `worker_count` agar kembali ke _worker_ pertama.

   - ```c
     buff.worker_number = index_worker;
     ```
     Berikutnya nilai `index_worker` disimpan ke `buff.worker_number`, yang menunjukkan nomor _worker_ tujuan.

   - ```c
     strcpy(buff.message, client_data->message);
     ```
     Pesan dari _client_ (`client_data->message`) disalin ke `buff.message` menggunakan `strcpy`.

   - ```c
     msgsnd(msgid, &buff, sizeof(MAX_STRING), 0);
     ```
     Fungsi `msgsnd(msgid, &buff, sizeof(MAX_STRING), 0)` akan dipanggil untuk mengirim pesan ke _message queue_. `msgsnd` mengirim data ke _queue_ dengan ID `msgid`, data yang dikirim adalah alamat dari `buff`, ukuran pesan yang dikirim adalah `sizeof(MAX_STRING)`, dan `flag 0` yang menunjukkan operasi diblokir jika _queue_ penuh.

   - ```c
     index_worker++;
     ```
     Setelah satu pesan dikirim, `index_worker` ditambah satu agar pesan berikutnya dikirim ke _worker_ selanjutnya.

```c
shmdt(shared_memory_for_worker);
shmdt(client_message);
```
Fungsi `shmdt` adalah fungsi yang digunakan untuk melepaskan _shared memory_ yang sebelumnya telah dipetakan. Fungsi ini memastikan bahwa proses tidak lagi mengakses _shared memory_ baik dari struktur data `shared_memory_for_worker` maupun `client_message`.

```c
return 0;
```
Kode diakhiri oleh `return 0` yang menandakan bahwa program telah berhasil dieksekusi tanpa _error_.

### Foto Hasil Output

![image alt](https://github.com/SuryaAndyartha/laporanmodul2_2/blob/main/Screenshot%202025-04-29%20at%2007.08.11.png?raw=true)

### c. Worker Mencatat Pesan yang Diterima 
### d. Catat Total Pesan yang Diterima Setiap Worker di Akhir Eksekusi (worker.c); Code Lengkap:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/msg.h>

#define MAX_STRING 512

typedef struct {
    int worker_count;
    int message_count;
} Data_for_Message_Queue; 

typedef struct {
    long int worker_number;
    char message[MAX_STRING];
} Message_for_Worker;

int main()
{

    // Shared Memory untuk jumlah worker
    key_t key = 4321;
    int shmid = shmget(key, sizeof(Data_for_Message_Queue), IPC_CREAT | 0666);
    if(shmid == -1){
        printf("shmget gagal.\n");
        return 1;
    }

    void* shared_memory_for_worker = shmat(shmid, NULL, 0);
    if(shared_memory_for_worker == (void *)-1){
        printf("shmat gagal.\n");
        return 2;
    }

    Data_for_Message_Queue *data = (Data_for_Message_Queue*) shared_memory_for_worker;

    int message_count = data->message_count;
    int worker_count = data->worker_count;

    // Message Queue untuk worker
    FILE *log_file = fopen("sistem.log", "a");
    if(log_file == NULL){
        printf("fopen gagal.\n");
        return 3;
    }

    int counter_message_received[worker_count];
    for (int i = 0; i < worker_count; i++) // Init
        counter_message_received[i] = 0;

    key = 2143;
    int msgid = msgget(key, 0666 | IPC_CREAT);
    int index_worker = 1;
    Message_for_Worker msg;
    for (int i = 0; i < message_count; i++)
    {
        if (index_worker > worker_count) index_worker -= worker_count;

        msgrcv(msgid, &msg, sizeof(MAX_STRING), index_worker, 0);
        if (msg.worker_number != index_worker)
        {
            printf("Index worker mismatch\n");
            return -1;
        }

        fprintf(log_file, "Worker%d: message received\n", index_worker);

        counter_message_received[index_worker-1]++;

        index_worker++;
    }

    for (int i = 0; i < worker_count; i++)
    {
        fprintf(log_file, "Worker %d: %d messages\n", i+1, counter_message_received[i]);
    }

    fclose(log_file);
    msgctl(msgid, IPC_RMID, NULL);
    shmdt(shared_memory_for_worker);
    shmctl(shmid, IPC_RMID, NULL);

    return 0;
}
```

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/msg.h>
```
Kode dimulai oleh beberapa _library_ penting untuk menjalankan program sesuai yang diinginkan. `#include <stdio.h>` menyediakan fungsi `printf`, `fprintf`, dan `fopen`. `#include <stdlib.h>` menyediakan fungsi `exit`. `#include <unistd.h>` menyediakan fungsi `shmdt`. `#include <sys/ipc.h>` menyediakan tipe data `key_t`. `#include <sys/shm.h>` menyediakan fungsi `shmget`, `shmat`, `shmdt`, dan `shmctl`. `#include <sys/msg.h>` menyediakan fungsi `msgget`, `msgrcv`, dan `msgctl`.

```c
#define MAX_STRING 512
```
Bagian ini mendefinisikan ukuran maksimal _array_ karakter yang akan digunakan untuk menyimpan _string_ sebagai 512 dengan `#define MAX_STRING 512`.

```c
typedef struct {
    int worker_count;
    int message_count;
} Data_for_Message_Queue; 
```
Di bagian ini, kode akan membuat sebuah `struct` dengan `typedef struct Data_for_Message_Queue`, yang berfungsi untuk mengelompokkan dua data: `worker_count`, yaitu variabel bertipe `int` untuk menyimpan jumlah _worker_ yang tersedia, dan `message_count` yang merupakan variabel bertipe `int` juga untuk menyimpan total jumlah pesan dari _client_ yang didapatkan melalui _loadbalancer_.

```c
typedef struct {
    long int worker_number;
    char message[MAX_STRING];
} Message_for_Worker;
```
Di bagian ini, kode akan membuat sebuah `struct` dengan `typedef struct Message_for_Worker`, yang berfungsi untuk mengelompokkan dua data: `worker_number`, yaitu variabel bertipe `long int` untuk menunjukkan nomor _worker_ yang akan dituju, dan `message`, yaitu  _array_ karakter berukuran `MAX_STRING` untuk menyimpan isi pesan telah dikirim ke _worker_.

```c
int main()
{

    // Shared Memory untuk jumlah worker
    key_t key = 4321;
    int shmid = shmget(key, sizeof(Data_for_Message_Queue), IPC_CREAT | 0666);
    if(shmid == -1){
        printf("shmget gagal.\n");
        return 1;
    }
    ...
}
```
`int main(){...}` adalah fungsi utama yang menjadi titik masuk eksekusi program. Di dalamnya, program dimulai dan dieksekusi.
Setelahnya, terdapat bagian yang akan mendeklarasikan sebuah variabel `key` bertipe `key_t` dan memberinya nilai `4321` untuk mengakses _shared memory_ antara _load balancer_ dan _worker_. Berikutnya ada fungsi `shmget` yang memiliki tiga parameter. Parameter pertama adalah `key`. Parameter kedua adalah `sizeof(Data_for_Message_Queue)`, yang menentukan ukuran dari _shared memory_, dalam kasus ini ukuran `Data_for_Message_Queue`. Parameter ketiga adalah `flag`, yaitu `IPC_CREAT` untuk membuat _shared memory_ jika belum ada dan `0666` untuk menentukan izin akses (_read and write_). Jika `shmget` gagal, maka `shmid` akan bernilai `-1`, sehingga program akan mencetak pesan `"shmget gagal."` dan mengembalikan nilai `1` sebagai tanda kegagalan.


```c
void* shared_memory_for_worker = shmat(shmid, NULL, 0);
if(shared_memory_for_worker == (void *)-1){
    printf("shmat gagal.\n");
    return 2;
}
```
`void* shared_memory_for_worker` adalah sebuah _pointer_ bertipe `void*` (tipe ini mampu menyimpan berbagai tipe data) yang digunakan untuk menyimpan alamat memori dari _shared memory_ antara _load balancer_ dan _worker_. Variabel ini akan menyimpan nilai yang dikembalikan oleh fungsi `shmat` dengan tiga parameter. Parameter pertama adalah `shmid`, yaitu ID _shared memory_. Parameter kedua adalah `NULL` yang berarti sistem akan memilih alamat yang sesuai untuk pemetaan. Parameter ketiga adalah `flag`, yang diatur ke 0 untuk pemetaan standar. Jika gagal, `shmat` akan mengembalikan `(void *)-1`, sehingga program akan mencetak pesan `"shmat gagal."` dan mengembalikan `nilai 2` sebagai tanda kegagalan. (Nilai `return` yang berbeda hanya digunakan untuk memudahkan pemeriksaan kesalahan/_debugging_).

```c
Data_for_Message_Queue *data = (Data_for_Message_Queue*) shared_memory_for_worker;
```
`Data_for_Message_Queue *data = (Data_for_Message_Queue*) shared_memory_for_worker` adalah konversi (_casting_) _pointer_ `shared_memory_for_worker` bertipe `void*` ke tipe _pointer_ `Data_for_Message_Queue*`. Ini memungkinkan akses langsung ke `struct Data_for_Message_Queue` yang ada di dalam _shared memory_.

```c
int message_count = data->message_count;
int worker_count = data->worker_count;
```
Bagian ini menyimpan informasi penting ke dalam _shared memory_ yang akan dioperasikan oleh proses _worker_. Nilai `message_count` dari _loadbalancer_ disimpan ke `data->message_count`, dan jumlah _worker_ yang dimasukkan _user_ melalui transfer _shared memory_ disimpan ke `data->worker_count`.

```c
FILE *log_file = fopen("sistem.log", "a");
if(log_file == NULL){
    printf("fopen gagal.\n");
    return 3;
}
```
`FILE *log_file = fopen("sistem.log", "a")` akan membuka _log file_ bernama `sistem.log` dalam mode _append_ (`"a"`) menggunakan fungsi `fopen`. Mode _append_ berarti data baru akan ditambahkan ke akhir _file_ jika _file_ tersebut sudah ada. Jika _file_ tidak ada, maka _file_ baru akan dibuat. Jika _log file_ gagal dibuka (`if(log_file == NULL)`), akan dicetak pesan `"fopen gagal."` dan program mengembalikan nilai dengan `return 3` sebagai tanda kegagalan. 

```c
int counter_message_received[worker_count];
for (int i = 0; i < worker_count; i++) // Init
    counter_message_received[i] = 0;
```
Bagian ini digunakan untuk mendeklarasikan dan menginisialisasi _array_ `counter_message_received` yang berfungsi menyimpan jumlah pesan yang diterima oleh masing-masing _worker_. _Array tersebut memiliki panjang sebesar jumlah worker (worker_count)_, dan setiap indeks diatur ke `0` dengan _looping_ `for`, menandakan bahwa belum ada pesan yang diterima pada awal program.

```c
key = 2143;
int msgid = msgget(key, 0666 | IPC_CREAT);
int index_worker = 1;
Message_for_Worker msg;
```
Bagian ini bertujuan untuk mendapatkan _message queue_ yang akan diterima oleh masing-masing _worker_. Pertama, `key` diatur ke `2143` (hanya sebagai pembeda). Setelah itu, fungsi `msgget(key, 0666 | IPC_CREAT)` dipanggil dan hasilnya disimpan dalam `msgid`. `msgget` memiliki dua parameter: parameter pertama adalah `key` dan parameter kedua adalah `flag` yang terdiri dari `0666` (izin _read_ and _write_) serta `IPC_CREAT` untuk membuat _message queue_ jika belum ada. Terakhir, `index_worker` diatur ke `1` sebagai penanda awal _worker_ yang akan menerima pesan pertama.

```c
for (int i = 0; i < message_count; i++)
{
    ...
}
```
Bagian ini merupakan proses distribusi dan pencatatan pesan yang diterima oleh masing-masing _worker_ menggunakan _looping_ `for` yang akan dijalankan sebanyak `message_count`, yaitu jumlah total pesan yang dikirim. Di dalam _loop_, akan dilakukan:

   - ```c
     if (index_worker > worker_count) index_worker -= worker_count;
     ```
     Jika `index_worke`r melebihi jumlah _worker_, maka dikurangi dengan `worker_count` agar kembali ke urutan worker pertama (metode _round-robin_).

   - ```c
     msgrcv(msgid, &msg, sizeof(MAX_STRING), index_worker, 0);
     ```
     Fungsi `msgrcv(msgid, &msg, sizeof(MAX_STRING), index_worker, 0)` dipanggil untuk menerima pesan dari _message queue_ dengan ID `msgid`. Parameter pertama adalah `msgid` yaitu ID dari _message queue_ dan parameter kedua adalah `&msg` yang merupakan _pointer_ ke _buffer_ tempat pesan disimpan. Parameter ketiga adalah `sizeof(MAX_STRING)`, yaitu ukuran data yang ingin diterima. Paramter keempat adalah `index_worker` atau tipe pesan yang ingin diterima dan terakhir merupakan `flag` yang diatur ke `0` dengan arti akan terjadi _blocking_ (menunggu sampai ada pesan).

   - ```c
     if (msg.worker_number != index_worker)
     {
        printf("Index worker mismatch\n");
        return -1;
     }
     ```
     Terjadi pemeriksaan apakah `msg.worker_number` sama dengan `index_worker` untuk memastikan bahwa pesan tersebut benar-benar ditujukan untuk _worker_ yang tepat. Jika tidak, program akan menampilkan pesan _error_ dan berhenti.

   - ```c
     fprintf(log_file, "Worker%d: message received\n", index_worker);
     ```
     Jika cocok, maka program mencatat di _log file_ bahwa _Worker_ ke-`i` telah menerima pesan.

   - ```c
     counter_message_received[index_worker-1]++;
     ```
     Nilai pada `counter_message_received[index_worker-1]` ditambahkan satu untuk mencatat jumlah pesan yang diterima oleh _worker_ tersebut.

   - ```c
     index_worker++;
     ```
     `index_worker` kemudian ditambah untuk pindah ke _worker_ berikutnya di iterasi selanjutnya.

```c
for (int i = 0; i < worker_count; i++)
{
    fprintf(log_file, "Worker %d: %d messages\n", i+1, counter_message_received[i]);
}
```
Di bagian ini, program akan mencetak jumlah pesan yang diterima oleh masing-masing _worker_ ke dalam _log file_. Dengan menggunakan _looping_ `for`, program akan menelusuri setiap indeks _array_ `counter_message_received` (yang menyimpan jumlah pesan setiap worker), lalu mencetaknya ke _file_ dengan format yang sudah ditentukan. `i+1` digunakan agar nomor _worker_ dimulai dari `1`, bukan `0`.

```c
fclose(log_file);
msgctl(msgid, IPC_RMID, NULL);
shmdt(shared_memory_for_worker);
shmctl(shmid, IPC_RMID, NULL);
```
Bagian ini bertujuan untuk memastikan tidak ada kebocoran sumber daya atau hal yang tidak diinginkan lainnya. `fclose(log_file)` akan menutup _log file_ yang sebelumnya dibuka. `msgctl(msgid, IPC_RMID, NULL)` akan menghapus _message queue_ dari sistem menggunakan ID `msgid`. `shmdt(shared_memory_for_worker)` akan melepas _shared memory_ dari proses saat ini.
`shmctl(shmid, IPC_RMID, NULL)` akan menghapus _shared memory_ dari sistem.

```c
return 0;
```
Kode diakhiri oleh `return 0` yang menandakan bahwa program telah berhasil dieksekusi tanpa _error_.

### Foto Hasil Output

![image alt](https://github.com/SuryaAndyartha/laporanmodul2_2/blob/main/Screenshot%202025-04-29%20at%2007.08.38.png?raw=true)

---
