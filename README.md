# Лабораторная работа 2 — Процессы и файловая система /proc

**Операционная система:** Ubuntu (через multipass на Mac M1)
**Автор:** Петраков Семён
**Группа:** 11A
**Дата выполнения:** 13.10.2025 


## Цель работы
Понять модель процессов Linux, принципы порождения и ожидания завершения, а также научиться извлекать информацию из виртуальной файловой системы `/proc`.

## Ход работы

### 1) Создание процессов

Написана программа на C, в которой процесс-родитель создаёт двух дочерних процессов. Каждый дочерний процесс выводит свой PID и PPID, а родитель ожидает их завершения.

**Код программы:**
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t child1, child2;
    int status;

    printf("parent: PID=%d\n", getpid());
    fflush(stdout);

    child1 = fork();
    if (child1 == 0) {
        printf("child_A: PID=%d, PPID=%d\n", getpid(), getppid());
        fflush(stdout);
        return 0;
    }

    child2 = fork();
    if (child2 == 0) {
        printf("child_B: PID=%d, PPID=%d\n", getpid(), getppid());
        fflush(stdout);
        return 0;
    }

    waitpid(child1, &status, 0);
    waitpid(child2, &status, 0);
    printf("parent: все дочерние процессы завершены\n");
    fflush(stdout);

    return 0;
}
```

**Вывод программы:**
![alt text](<Screenshot 2025-10-20 at 11.04.41.png>)
---

### 2) Исследование дерева процессов

Выполнены команды для визуализации дерева процессов:

```bash
ps -ef --forest | head -n 30
pstree -p | head -n 50
```
![alt text](<Screenshot 2025-10-20 at 11.04.52.png>)
![alt text](<Screenshot 2025-10-20 at 11.05.08.png>)

---

### 3) Изучение /proc

Изучены файлы в `/proc` для текущей оболочки (PID=`$$`)

- `/proc/<pid>/cmdline` — командная строка запуска процесса.
- `/proc/<pid>/status` — информация о состоянии процесса: UID, PPID, статус, память и т.д.
- `/proc/<pid>/fd` — список открытых файловых дескрипторов.

![alt text](<Screenshot 2025-10-20 at 11.05.30.png>)
![alt text](<Screenshot 2025-10-20 at 11.05.53.png>)
![alt text](<Screenshot 2025-10-20 at 11.06.07.png>)
![alt text](<Screenshot 2025-10-20 at 11.06.07.png>)
![alt text](<Screenshot 2025-10-20 at 11.06.21.png>)
![alt text](<Screenshot 2025-10-20 at 11.06.32.png>)
---

### 4) Анализ процессов (нагрузка CPU/память/IO)

#### Топ процессов по CPU:
```bash
ps -eo pid,ppid,comm,state,%cpu,%mem,etime --sort=-%cpu | head -n 6
```
![alt text](<Screenshot 2025-10-20 at 11.06.51.png>)

#### Топ процессов по памяти:
```bash
ps -eo pid,ppid,comm,state,%mem,rss --sort=-%mem | head -n 6
```
![alt text](<Screenshot 2025-10-20 at 11.07.01.png>)

#### Нагрузка за интервал:
```bash
pidstat -u -r -d 1 5
```
![alt text](<Screenshot 2025-10-20 at 11.07.43.png>)

---

### 5) Мини-утилита ptree

Написана утилита на c, которая выводит цепочку родительских процессов от текущего до init/systemd:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <dirent.h>

// Функция для получения имени процесса по PID
char* get_process_name(pid_t pid) {
    static char name[256];
    char path[256];
    FILE* file;
    
    snprintf(path, sizeof(path), "/proc/%d/comm", pid);
    file = fopen(path, "r");
    
    if (file) {
        if (fgets(name, sizeof(name), file)) {
            // Убираем символ новой строки
            name[strcspn(name, "\n")] = 0;
        }
        fclose(file);
    } else {
        snprintf(name, sizeof(name), "unknown");
    }
    
    return name;
}

// Функция для получения PPID
pid_t get_ppid(pid_t pid) {
    char path[256];
    FILE* file;
    char line[256];
    pid_t ppid = 0;
    
    snprintf(path, sizeof(path), "/proc/%d/status", pid);
    file = fopen(path, "r");
    
    if (file) {
        while (fgets(line, sizeof(line), file)) {
            if (strncmp(line, "PPid:", 5) == 0) {
                sscanf(line + 5, "%d", &ppid);
                break;
            }
        }
        fclose(file);
    }
    
    return ppid;
}

int main() {
    pid_t current_pid = getpid();
    pid_t ppid;
    
    printf("Цепочка процессов от PID=%d до init:\n", current_pid);
    
    while (current_pid > 1) {
        printf("%s(%d)", get_process_name(current_pid), current_pid);
        ppid = get_ppid(current_pid);
        
        if (ppid > 0) {
            printf(" -> ");
            current_pid = ppid;
        } else {
            break;
        }
    }
    
    printf("%s(1)\n", get_process_name(1));
    return 0;
}
```

**Пример вывода:**
![alt text](<Screenshot 2025-10-20 at 11.14.09.png>)
---

## Ответы на вопросы

1. **Процесс** — это выполняющийся экземпляр программы с контекстом (память, регистры, файлы), а **программа** — это статичный исполняемый файл.
2. Если вызвать `fork()` без `wait()`, дочерний процесс станет **зомби** после завершения, пока родитель не прочитает его статус.
3. Система хранит информацию о процессах в виртуальной файловой системе `/proc`.
4. `exec()` заменяет образ текущего процесса новым, загружая другую программу.
5. В `/proc` нет настоящих файлов — это виртуальная ФС, отражающая состояние ядра.
6. В `top`:
   - `%CPU` — использование CPU;
   - `%MEM` — использование памяти;
   - `VIRT` — виртуальная память;
   - `RES` — резидентная память;
   - `SHR` — разделяемая память;
   - `TIME+` — общее время CPU.
7. Сумма `%CPU` может быть >100% на многопроцессорных системах (одно ядро = 100%).
8. **Мгновенное %CPU** — текущая нагрузка, **load average** — средняя нагрузка за 1, 5, 15 мин. `wa` — время ожидания ввода-вывода.
9. **IO-нагрузка** — активность диска, **CPU-нагрузка** — использование процессора. Смотреть можно через `pidstat -d`, `iotop`, `/proc/<pid>/io`.
10. **Nice** — приоритет планировщика. Чем выше nice, тем ниже приоритет.
11. **Поток** — часть процесса, разделяет память. В `ps` показываются с ключом `-L`, в `top` — `H`.
12. **Зомби** — завершённые процессы, чей статус не прочитан. **Сироты** — процессы, чей родитель завершился. Их усыновляет `init`.

---

## Выводы
В ходе работы изучены механизмы создания и управления процессами в Linux, исследована файловая система `/proc`, проанализирована нагрузка на систему и написана утилита для отображения цепочки процессов.