# postgresql_postgis_crash
## crash postgres on create index

## How to restore crash:
1. Create DB:
    ```
    psql -p 5432 -c "create database test_crash"
    ```

2. Add postgis: 
    ```
    psql -p 5432 -c "create extension postgis" -d "test_crash"
    ```

3. Check versions:
    - Postgresql:
        ```
        psql -p 5432 -c "select version()" -d "test_crash" 
        ```
        return:
        ```
        PostgreSQL 10.12 (Ubuntu 10.12-2.pgdg16.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0-6ubuntu1~16.04.12) 5.4.0 20160609, 64-bit
        ```

    - Postgis:
        ```
        psql -p 5432 -c "select postgis_full_version()" -d "test_crash"
        ```
        return:
        ```
        POSTGIS="2.5.3 r17699" [EXTENSION] PGSQL="100" GEOS="3.7.1-CAPI-1.11.1 27a5e771" PROJ="Rel. 4.9.2, 08 September 2015" GDAL="GDAL 1.11.3, released 2015/09/16" LIBXML="2.9.3" LIBJSON="0.11.99" LIBPROTOBUF="1.2.1" RASTER
        ```

5. Import data to DB:
    ```
    psql -p 5432 -f "./test_table.sql" -d "test_crash" 
    ```
6. Exec SQL:
    ```
    psql -p 5432 -c "drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;" -d "test_crash"
    ```
    return:
    ```
    server closed the connection unexpectedly
            This probably means the server terminated abnormally
            before or while processing the request.
    connection to server was lost                   
    ```
    Postgresql log file info:
    ```
    2020-02-28 14:40:08.170 CET [11812] DZIENNIK:  proces serwera (PID 25639) został zatrzymany przez sygnał 11: Segmentation fault       
    2020-02-28 14:40:08.170 CET [11812] SZCZEGÓŁY:  Zawiódł wykonywany proces: drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;                          
    2020-02-28 14:40:08.170 CET [11812] DZIENNIK:  kończenie wszelkich innych aktywnych procesów serwera                                  
    2020-02-28 14:40:08.170 CET [15770] OSTRZEŻENIE:  zakończenie połączenia spowodowane awarią innego procesu serwera                    
    2020-02-28 14:40:08.170 CET [15770] SZCZEGÓŁY:  Postmaster nakazał temu procesowi serwera wycofanie bieżącej transakcji i wyjście, gdyż inny proces serwera zakończył się nieprawidłowo i pamięć współdzielona może być uszkodzona.                                         
    2020-02-28 14:40:08.170 CET [15770] PODPOWIEDŹ:  Za chwilę będziesz mógł połączyć się ponownie do bazy danych i powtórzyć polecenie.  
    2020-02-28 14:40:08.183 CET [11812] DZIENNIK:  wszelkie procesy serwera zakończone; ponowna inicjacja                                 
    2020-02-28 14:40:08.441 CET [25649] DZIENNIK:  działanie systemu bazy danych zostało przerwane; ostatnie znane podniesienie 2020-02-28 14:30:14 CET                                                                                                                         
    2020-02-28 14:40:08.478 CET [25649] DZIENNIK:  system bazy danych nie został poprawnie zamknięty; trwa automatyczne odzyskiwanie      
    2020-02-28 14:40:08.481 CET [25649] DZIENNIK:  ponowienie uruchamia się w 0/26144078                                                  
    2020-02-28 14:40:08.481 CET [25649] DZIENNIK:  niepoprawna długość rekordu w 0/261440B0: oczekiwana 24, jest 0                        
    2020-02-28 14:40:08.481 CET [25649] DZIENNIK:  ponowienie wykonane w 0/26144078                                                       
    2020-02-28 14:40:08.571 CET [11812] DZIENNIK:  system bazy danych jest gotowy do przyjmowania połączeń           
    ```

## Server 
- ``` lsb_release -a ``` return:
    ```
        No LSB modules are available.
        Distributor ID:	Ubuntu
        Description:	Ubuntu 16.04.6 LTS
        Release:	16.04
        Codename:	xenial
    ```

- ``` uname -a ``` return: 
    ```
        Linux 4.4.0-165-generic #193-Ubuntu SMP Tue Sep 17 17:42:52 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
    ```


## System crash dump
I don't know how to restore errors below but on other db I had system crash file. 
Belowe some info aobut that.
* cd ~/; mkdir ./crash/err -p; cd ./crash; mv /var/crash/_usr_lib_postgresql_10_bin_postgres.111.crash ~/crash 
* apport-unpack _usr_lib_postgresql_10_bin_postgres.111.crash ./err; cd ./err
* gdb /usr/lib/postgresql/10/bin/postgres --core=./CoreDump
 - (gdb) bt full
    ```
    #0  0x000055fb9015c45b in pfree ()
    No symbol table info available.
    #1  0x00007f64ee854b5e in ?? () from /usr/lib/postgresql/10/lib/postgis-2.5.so
    No symbol table info available.
    #2  0x00007f64ee856367 in gserialized_gist_picksplit () from /usr/lib/postgresql/10/lib/postgis-2.5.so
    No symbol table info available.
    #3  0x000055fb9013a272 in FunctionCall2Coll ()
    No symbol table info available.
    #4  0x000055fb8fd7f526 in gistSplitByKey ()
    No symbol table info available.
    #5  0x000055fb8fd7652c in gistSplit ()
    No symbol table info available.
    #6  0x000055fb8fd76952 in gistplacetopage ()
    No symbol table info available.
    #7  0x000055fb8fd7741e in ?? ()
    No symbol table info available.
    #8  0x000055fb8fd7782a in gistdoinsert ()
    No symbol table info available.
    #9  0x000055fb8fd80d11 in ?? ()
    No symbol table info available.
    #10 0x000055fb8fdf55e7 in IndexBuildHeapRangeScan ()
    No symbol table info available.
    #11 0x000055fb8fdf5b0c in IndexBuildHeapScan ()
    No symbol table info available.
    #12 0x000055fb8fd81380 in gistbuild ()
    No symbol table info available.
    #13 0x000055fb8fdf6923 in index_build ()
    No symbol table info available.
    #14 0x000055fb8fdf82b9 in index_create ()
    No symbol table info available.
    #15 0x000055fb8fe958b0 in DefineIndex ()
    No symbol table info available.
    #16 0x000055fb90026f80 in ?? ()
    No symbol table info available.
    #17 0x000055fb90025f87 in standard_ProcessUtility ()
    No symbol table info available.
    #18 0x000055fb90022fab in ?? ()
    No symbol table info available.
    #19 0x000055fb90023a98 in ?? ()
    No symbol table info available.
    #20 0x000055fb90024805 in PortalRun ()
    No symbol table info available.
    #21 0x000055fb900202bc in ?? ()
    No symbol table info available.
    #22 0x000055fb900217e8 in PostgresMain ()
    No symbol table info available.
    #23 0x000055fb8fd5587e in ?? ()
    No symbol table info available.
    #24 0x000055fb8ffaf819 in PostmasterMain ()
    No symbol table info available.
    #25 0x000055fb8fd56c75 in main ()
    No symbol table info available.
    ```

 - (gdb) bt full
    ```
    #0  0x000055fb9015c45b in pfree ()
    No symbol table info available.
    #1  0x00007f64ee7f2b5e in ?? () from /usr/lib/postgresql/10/lib/postgis-2.5.so
    No symbol table info available.
    #2  0x00007f64ee7f4367 in gserialized_gist_picksplit () from /usr/lib/postgresql/10/lib/postgis-2.5.so
    No symbol table info available.
    #3  0x000055fb9013a272 in FunctionCall2Coll ()
    No symbol table info available.
    #4  0x000055fb8fd7f526 in gistSplitByKey ()
    No symbol table info available.
    #5  0x000055fb8fd7652c in gistSplit ()
    No symbol table info available.
    #6  0x000055fb8fd76952 in gistplacetopage ()
    No symbol table info available.
    #7  0x000055fb8fd7741e in ?? ()
    No symbol table info available.
    #8  0x000055fb8fd7782a in gistdoinsert ()
    No symbol table info available.
    #9  0x000055fb8fd781e7 in gistinsert ()
    No symbol table info available.
    #10 0x000055fb8fee4aa4 in ExecInsertIndexTuples ()
    No symbol table info available.
    #11 0x000055fb8ff04b4a in ?? ()
    No symbol table info available.
    #12 0x000055fb8fee5c0b in standard_ExecutorRun ()
    No symbol table info available.
    #13 0x000055fb900238f5 in ?? ()
    No symbol table info available.
    #14 0x000055fb90023b4c in ?? ()
    No symbol table info available.
    #15 0x000055fb90024805 in PortalRun ()
    No symbol table info available.
    #16 0x000055fb900202bc in ?? ()
    No symbol table info available.
    #17 0x000055fb900217e8 in PostgresMain ()
    No symbol table info available.
    #18 0x000055fb8fd5587e in ?? ()
    No symbol table info available.
    #19 0x000055fb8ffaf819 in PostmasterMain ()
    No symbol table info available.
    #20 0x000055fb8fd56c75 in main ()
    No symbol table info available.
    ```