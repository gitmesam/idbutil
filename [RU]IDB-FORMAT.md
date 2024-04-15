
IDApro databases
==================

База данных IDApro состоит из одного большого файла, который содержит несколько разделов. 
В начале файла idb или i64 находится список смещений файлов, указывающих на эти разделы. 
При желании разделы могут храниться в сжатом виде. 
Когда IDA открывает базу данных, разделы извлекаются из основного файла данных.
и хранятся в отдельных файлах. Когда вам нужно только читать из базы данных, а не
хотите что-то изменить, разделение на файлы `id0`,` id1`, `nam` и` til` вовсе не является
обязательным, однако, IDApro все равно делает это, поскольку ожидает, что пользователь внесет изменения в базу данных.
Очень старые версии IDApro (v1.6 и v2.0) хранят разделы отдельно, так что
в каждом каталоге может быть только одна база данных.

| index | extension | contents                   |
| :---- | :-------- | :------------------------- |
|   0   |  id0      | A btree key/value database |
|   1   |  id1      | Flags for each byte        |
|   2   |  nam      | A list of named offsets    |
|   3   |  seg      | Unknown                    |
|   4   |  til      | Type information           |
|   5   |  id2      | Unknown                    |

Старые версии Ida - не имеют id2 file.
А более новые - не имеют seg file.
Новые версии Ida используют 64 bit file offsets, таким образом, IDA может поддерживать файлы с размером более чем 4GB.
Не существует разницы в заголовке IDB между 32-битными (с расширением .idb) и 64-битными (с расширением .i64) базами данных.

## ID0 section.

Секция ID0 содержит b-tree database, Это просто большая база key-value, like leveldb.
Есть три группы key types:

 * Bookkeeping, so IDApro can quickly decide what the next free nodeid is. These keys all start with a '$' (dollar) sign.
    * `$ MAX LINK`
    * `$ MAX NODE`
    * `$ NET DESC`
 * Nodes, keys начинающиеся с '.' (dot).
    * стоящие перед address, или internal nodeid.
       * 32 bit базы используют 32 bit адресы, 64 bit базы используют 64 bit адресы.
       * internal nodeid's имеют старшие 8 bits установленными в 1,
         таким образом, `0xFF000000` для 32 bit баз, или `0xFF00000000000000` для 64 bit баз.
    * tag,  `A` для altvals, `S` для supvals, и т. д. См. netnode.hpp в idasdk.
    * стоящие перед index или hashkey value, взависимости от tag.
    * both the address and index value are encoded in bigendian byte order.
 * Name index, keys именуются `N`, стоящие перед name.
    value могут быть 32 или 64 bit смещениями.
    * names больше 511 кодируются, как plain strings. longer names начинаются с NUL byte, стоящим перед blob index.
      указывающим на blob со специальным nodeid `0xFF000000(00000000)`.
    * maximum name length 32 * 1024 символов.
 * Очень старые версии ida имеют keys начинающиеся со строчной 'n', и '-' (minus).
 * maximum key size если они 512 bytes, включают точку, 'N', и т. п.

Диапазон internal nodeid's - причина по которой вы не можете иметь дизассемблированный code или data 
по адресам, начиная с `0xFF000000(00000000)`. IDA позволяет это сделать - вручную. 
Как правило, это приводит к повреждению баз данных.
Есть 2 типа names:
 * Internal, указывающие на internal nodeid's. Examples: `$ structs`, `Root Node`. Most have a space in them.
 * Labels, указывающие на addresses в дизассемблере.

maximum value size равен 1024 bytes.
Различные типы values:
 * Integers, кодируются в little endian byte order.
 * Strings иногда NUL terminated, иногда нет.
 * Иногда информация упакована в _packed_ format, см. ниже.

### packed values

Упакованные значения используются, среди прочего, для определений структуры и сегментов. 

В упакованных данных:
 * Values в диапазоне 0x00-0x7f хранятся, как single byte.
 * Values в диапазоне 0x80-0x3fff хранятся, как ORRED with 0x8000.
 * Values в диапазоне 0x4000-0x1fffffff хранятся, как ORRED with 0xC000000.
 * Старшие 32 bit values хранятся, как prefixed with a 0xFF byte.
 * 64 bit values хранятся, как два подряд идущих числа. 
 * Все values хранятся в big-endian byte order.


### The B-tree format

Файл организован в виде страниц размером 8 Кбайт, где первая страница содержит заголовок с
 * размером страницы, 
 * указатель на список свободных страниц, 
 * указатель на корень дерева страниц,
 * количество записей и количество страниц. 

Есть два типа страниц: 
 * листовые страницы, которые не содержат указателей на другие страницы, а только записи типа "ключ-значение". 
 * И индексные страницы с _preceeding_ указатель, и где все записи "ключ-значение" содержат указатель на страницу, где все
ключи на указанной странице имеют значения больше, чем ключ, содержащий
указатель страницы. Это делает очень эффективным поиск записей по ключу.

Page tree выглядит так. В скобках указаны ключевые значения, указатель отмечен
со знаком `*` (ЗВЕЗДА) - это _preceeding_ указатель. Значения не показаны. 


                       *-------->[00]
             *------>[02]---+    [01]
    root ->[08]---+  [05]-+ |    
           [17]-+ |       | +--->[03]
                | |       |      [04]
                | |       |      
                | |       +----->[06]
                | |              [07]
                | |
                | |    *-------->[09]
                | +->[11]---+    [10]
                |    [14]-+ |  
                |         | +--->[12]
                |         |      [13]
                |         |
                |         +----->[15]
                |                [16]
                |       
                |      *-------->[18]
                +--->[20]---+    [19]
                     [23]-+ |  
                          | +--->[21]
                          |      [22]
                          |
                          +----->[24]
                                 [25]




Каждая страница имеет небольшой заголовок с указателем на предыдущую страницу и счетчик записей.
Для страниц Leaf предыдущий указатель равен нулю.

После заголовка идет индекс, содержащий смещения к фактическим записям на странице,
и указатель на индекс следующего уровня или конечную страницу. 
Записи хранятся как _keylength_, keydata, _datalength_, data.
Все записи на уровне ниже индекса гарантированно имеют ключ больше, чем ключ
в индексе. 

На конечных страницах последовательные записи часто имеют очень похожие ключи. Индекс хранит
смещение в ключе, от которого отличаются ключи, сохраняется только отличающаяся часть. 

| key                                | binary representation   | compressed key
| :--------------------------------- | :---------------------- | :------------------
| ('.', 0xFF000002, 'N')             | 2eff0000024e            | (0, 2eff0000024e)         
| ('.', 0xFF000002, 'S', 0x00000001) | 2eff0000025300000001    | (5, 5300000001)
| ('.', 0xFF000002, 'S', 0x00000002) | 2eff0000025300000002    | (9, 02)


## The ID1 section

Секция ID1 содержит значения флагов, возвращаемые функцией idc `GetFlags`.
Он начинается со списка областей файла, за которым следуют флаги для каждого байта.

## Netnodes

Высокоуровневое представление базы `ID0` это netnodes, как было частично 
документировано в idasdk.

Наиболее важные nodes это:
 * `Root Node`
 * lists: `$ structs`, `$ enums`, `$ scripts`
   * the values in a list are stored in the altnodes of the list node.
   * the values are one more than the actual nodeid pointed to:
     a list pointing to struct id's 0xff000bf6, 0xff000c01
     would contain : 0xff000bf7, 0xff000c02
 * `$ funcs`
 * `$ fileregions`, `$ segs`, '$ srareas'
 * '$ entry points'


### structs

The main struct node:

| node | contents
| :--- | :----
| (id, 'N')    | the struct name
| (id, 'M', 0) | packed member info, nodeids for members.


The struct member nodes:

| node | contents
| :--- | :----
| (id, 'N')    | the member name
| (id, 'M', 0) | packed member info
| (id, 'A', 3) | enum id
| (id, 'A', 11) | struct id
| (id, 'A', 16) | string type
| (id, 'S', 0) | member comment
| (id, 'S', 1) | repeatable member comment
| (id, 'S', 9) | offset spec
| (id, 'S', 0x3000) | typeinfo


### history

The `$ curlocs` list contains several location histories:

For example, the `$ IDA View-A` netnode contains the following keys:
 * `A 0` - highest history supval item 
 * `A 1` - number of history items
 * `A 2` - object type: `idaplace_t`
 * `S <num>` - packed history item: itemlinenr, ea\_t, int, int, colnr, rownr

 

### normal addresses

In the SDK, in the file `nalt.hpp` there are many more items defined.
These are some of the regularly used ones.

| key | value | description
| :-- | :---- | :----------
| (addr, 'D', fromaddr) | reftype | data xref from
| (addr, 'd', toaddr) | reftype | data xref to
| (addr, 'X', fromaddr) | reftype | code xref from
| (addr, 'x', toaddr) | reftype | code xref to
| (addr, 'N')         | string | global label
| (addr, 'A', 1)      | jumptableid+1 | jumptable target
| (addr, 'A', 2)      | nodeid+1 | hexrays info
| (addr, 'A', 3)      | structid+1  | data type
| (addr, 'A', 8)      | dword    | additional flags
| (addr, 'A', 0xB)    | enumid+1 | first operand enum type
| (addr, 'A', 0x10)   | dword  | string type
| (addr, 'A', 0x11)   | dword  | align type
| (addr, 'S', 0)      | string | comment
| (addr, 'S', 1)      | string | repeatable comment
| (addr, 'S', 4)      | data   | constant pool reference
| (addr, 'S', 5)      | data   | array
| (addr, 'S', 8)      | data   | jumptable info
| (addr, 'S', 9)      | packed | first operand offset spec
| (addr, 'S', 0xA)    | packed | second operand offset spec
| (addr, 'S', 0x1B)    | data  |  ?
| (addr, 'S', 1000+linenr) | string | anterior comment
| (addr, 'S', 0x1000) | packed | SP change point
| (addr, 'S', 0x3000) | data | function prototype
| (addr, 'S', 0x3001) | data | argument list
| (addr, 'S', 0x4000+n) | packed blob | register renaming
| (addr, 'S', 0x5000) | packed blob | function's local labels
| (addr, 'S', 0x6000) | data | register args
| (addr, 'S', 0x7000) | packed | function tails
| (addr, 'S', 0x7000) | dword  | tail backreference
| 
