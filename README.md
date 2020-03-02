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
I tested again - remove system dump core file, remove logs file and restart cluster.
Now I run above SQL killer query.
Results:

- dmesg
    ```
    traps: postgres[24881] general protection ip:55a5b619f45b sp:7ffefd93b910 error:0 in postgres[55a5b5ce5000+6e1000]
    ```
- system core dump file:
  - unpack (some error/msg ocurred but unpack is done ok):
    ```
        ./crash# mkdir err
        ./crash# apport-unpack _usr_lib_postgresql_10_bin_postgres.111.crash err/
        Traceback (most recent call last):
        File "/usr/bin/apport-unpack", line 73, in <module>
            pr.extract_keys(f, bin_keys, dir)
        File "/usr/lib/python3/dist-packages/problem_report.py", line 270, in extract_keys
            [item for item, element in b64_block.items() if element is False])
        ValueError: ['ProcEnviron'] has no binary content
        ./crash# 
    ```    

  - analize CoreDump (after install postgresql-10-c )
    ```
    ./crash# cd ./err

    ./crash/err# gdb /usr/lib/postgresql/10/bin/postgres CoreDump 
    ```
    return:
    - new crash dump file generated after install postgres dbg and run gbd -p 'postgres-10-PID'
        ```
        ./err# gdb /usr/lib/postgresql/10/bin/postgres CoreDump 
        GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
        Copyright (C) 2016 Free Software Foundation, Inc.
        License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
        This is free software: you are free to change and redistribute it.
        There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
        and "show warranty" for details.
        This GDB was configured as "x86_64-linux-gnu".
        Type "show configuration" for configuration details.
        For bug reporting instructions, please see:
        <http://www.gnu.org/software/gdb/bugs/>.
        Find the GDB manual and other documentation resources online at:
        <http://www.gnu.org/software/gdb/documentation/>.
        For help, type "help".
        Type "apropos word" to search for commands related to "word"...
        Reading symbols from /usr/lib/postgresql/10/bin/postgres...Reading symbols from /usr/lib/debug/.build-id/be/2d0e659760671fda0395f73e15ef668c2c4235.debug...done.
        done.
        [New LWP 19552]
        [Thread debugging using libthread_db enabled]
        Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
        Core was generated by `postgres: 10/test: postgres test_crash [local] CREATE INDEX                   '.
        Program terminated with signal SIGSEGV, Segmentation fault.
        #0  pfree (pointer=0x7f7a6e4068f8) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/mmgr/mcxt.c:954
        954	/build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/mmgr/mcxt.c: Nie ma takiego pliku ani katalogu.

        #0  pfree (pointer=0x7f7a6e4068f8) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/mmgr/mcxt.c:954
        #1  0x00007f7a65e39b5e in ?? () from /usr/lib/postgresql/10/lib/postgis-2.5.so
        #2  0x00007f7a65e3b367 in gserialized_gist_picksplit () from /usr/lib/postgresql/10/lib/postgis-2.5.so
        #3  0x000055b80d1f6272 in FunctionCall2Coll (flinfo=flinfo@entry=0x55b80ee4fb08, collation=<optimized out>, arg1=arg1@entry=94249012395288, arg2=arg2@entry=140736435822256) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/fmgr/fmgr.c:1059
        #4  0x000055b80ce3b526 in gistUserPicksplit (len=227, giststate=0x55b80ee4dce8, itup=0x55b80ee634c8, v=0x7fffc1439eb0, attno=0, entryvec=0x55b80ee77d18, r=0x7f7a660a10a8)
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistsplit.c:433
        #5  gistSplitByKey (r=r@entry=0x7f7a660a10a8, page=page@entry=0x7f7a6e404a00 "", itup=itup@entry=0x55b80ee634c8, len=len@entry=227, giststate=giststate@entry=0x55b80ee4dce8, v=v@entry=0x7fffc1439eb0, attno=0)
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistsplit.c:697
        #6  0x000055b80ce3252c in gistSplit (r=r@entry=0x7f7a660a10a8, page=page@entry=0x7f7a6e404a00 "", itup=0x55b80ee634c8, len=227, giststate=giststate@entry=0x55b80ee4dce8) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:1375
        #7  0x000055b80ce32952 in gistplacetopage (rel=0x7f7a660a10a8, freespace=<optimized out>, giststate=giststate@entry=0x55b80ee4dce8, buffer=520, itup=itup@entry=0x7fffc143a388, ntup=ntup@entry=1, oldoffnum=0, newblkno=0x0, leftchildbuf=0, splitinfo=0x7fffc143a2c0, 
            markfollowright=1 '\001', heapRel=0x7f7a6609f9b8) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:295
        #8  0x000055b80ce3341e in gistinserttuples (state=state@entry=0x7fffc143a390, stack=stack@entry=0x7fffc143a3b0, giststate=giststate@entry=0x55b80ee4dce8, tuples=tuples@entry=0x7fffc143a388, ntup=ntup@entry=1, oldoffnum=oldoffnum@entry=0, leftchild=0, rightchild=0, 
            unlockbuf=0 '\000', unlockleftchild=0 '\000') at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:1221
        #9  0x000055b80ce3382a in gistinserttuple (oldoffnum=0, tuple=0x55b80ee63498, giststate=0x55b80ee4dce8, stack=0x7fffc143a3b0, state=0x7fffc143a390) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:1180
        #10 gistdoinsert (r=r@entry=0x7f7a660a10a8, itup=itup@entry=0x55b80ee63498, freespace=<optimized out>, giststate=0x55b80ee4dce8, heapRel=<optimized out>) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:837
        #11 0x000055b80ce3cd11 in gistBuildCallback (index=index@entry=0x7f7a660a10a8, htup=htup@entry=0x55b80edad700, values=values@entry=0x7fffc143a590, isnull=isnull@entry=0x7fffc143a8e0 "", tupleIsAlive=tupleIsAlive@entry=1 '\001', state=state@entry=0x7fffc143a9a0)
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistbuild.c:488
        #12 0x000055b80ceb15e7 in IndexBuildHeapRangeScan (heapRelation=heapRelation@entry=0x7f7a6609f9b8, indexRelation=indexRelation@entry=0x7f7a660a10a8, indexInfo=indexInfo@entry=0x55b80eda6428, allow_sync=allow_sync@entry=1 '\001', anyvisible=anyvisible@entry=0 '\000', 
            start_blockno=start_blockno@entry=0, numblocks=4294967295, callback=0x55b80ce3cc80 <gistBuildCallback>, callback_state=0x7fffc143a9a0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:2701
        #13 0x000055b80ceb1b0c in IndexBuildHeapScan (heapRelation=heapRelation@entry=0x7f7a6609f9b8, indexRelation=indexRelation@entry=0x7f7a660a10a8, indexInfo=indexInfo@entry=0x55b80eda6428, allow_sync=allow_sync@entry=1 '\001', 
            callback=callback@entry=0x55b80ce3cc80 <gistBuildCallback>, callback_state=callback_state@entry=0x7fffc143a9a0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:2271
        #14 0x000055b80ce3d380 in gistbuild (heap=0x7f7a6609f9b8, index=0x7f7a660a10a8, indexInfo=0x55b80eda6428) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistbuild.c:207
        #15 0x000055b80ceb2923 in index_build (heapRelation=heapRelation@entry=0x7f7a6609f9b8, indexRelation=indexRelation@entry=0x7f7a660a10a8, indexInfo=indexInfo@entry=0x55b80eda6428, isprimary=isprimary@entry=0 '\000', isreindex=isreindex@entry=0 '\000')
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:2132
        #16 0x000055b80ceb42b9 in index_create (heapRelation=heapRelation@entry=0x7f7a6609f9b8, indexRelationName=indexRelationName@entry=0x55b80eda6e78 "test_table_geom_idx", indexRelationId=548864, indexRelationId@entry=0, relFileNode=<optimized out>, 
            indexInfo=indexInfo@entry=0x55b80eda6428, indexColNames=indexColNames@entry=0x55b80edadb08, accessMethodObjectId=783, tableSpaceId=1663, collationObjectId=0x55b80edadb50, classObjectId=0x55b80edadb68, coloptions=0x55b80edadb80, reloptions=0, isprimary=0 '\000', 
            isconstraint=0 '\000', deferrable=0 '\000', initdeferred=0 '\000', allow_system_table_mods=0 '\000', skip_build=0 '\000', concurrent=0 '\000', is_internal=0 '\000', if_not_exists=0 '\000')
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:1133
        #17 0x000055b80cf518b0 in DefineIndex (relationId=<optimized out>, relationId@entry=534024, stmt=stmt@entry=0x55b80eda6ea8, indexRelationId=indexRelationId@entry=0, is_alter_table=is_alter_table@entry=0 '\000', check_rights=check_rights@entry=1 '\001', 
            check_not_in_use=check_not_in_use@entry=1 '\001', skip_build=0 '\000', quiet=0 '\000') at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/commands/indexcmds.c:682
        #18 0x000055b80d0e2f80 in ProcessUtilitySlow (pstate=pstate@entry=0x55b80eda62c8, pstmt=pstmt@entry=0x55b80ee12690, 
            queryString=queryString@entry=0x55b80ee11358 "drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;", context=context@entry=PROCESS_UTILITY_QUERY, 
            params=params@entry=0x0, queryEnv=queryEnv@entry=0x0, completionTag=0x7fffc143b3f0 "", dest=0x55b80ee128d0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/utility.c:1336
        #19 0x000055b80d0e1f87 in standard_ProcessUtility (pstmt=0x55b80ee12690, 
            queryString=0x55b80ee11358 "drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;", context=PROCESS_UTILITY_QUERY, params=0x0, queryEnv=0x0, 
            dest=0x55b80ee128d0, completionTag=0x7fffc143b3f0 "") at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/utility.c:931
        #20 0x000055b80d0defab in PortalRunUtility (portal=0x55b80ed79d48, pstmt=0x55b80ee12690, isTopLevel=<optimized out>, setHoldSnapshot=<optimized out>, dest=<optimized out>, completionTag=0x7fffc143b3f0 "")
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/pquery.c:1178
        #21 0x000055b80d0dfa98 in PortalRunMulti (portal=portal@entry=0x55b80ed79d48, isTopLevel=isTopLevel@entry=0 '\000', setHoldSnapshot=setHoldSnapshot@entry=0 '\000', dest=dest@entry=0x55b80ee128d0, altdest=altdest@entry=0x55b80ee128d0, 
            completionTag=completionTag@entry=0x7fffc143b3f0 "") at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/pquery.c:1331
        #22 0x000055b80d0e0805 in PortalRun (portal=portal@entry=0x55b80ed79d48, count=count@entry=9223372036854775807, isTopLevel=isTopLevel@entry=0 '\000', run_once=run_once@entry=1 '\001', dest=dest@entry=0x55b80ee128d0, altdest=altdest@entry=0x55b80ee128d0, 
            completionTag=0x7fffc143b3f0 "") at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/pquery.c:799
        #23 0x000055b80d0dc2bc in exec_simple_query (query_string=0x55b80ee11358 "drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;")
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/postgres.c:1122
        #24 0x000055b80d0dd7e8 in PostgresMain (argc=<optimized out>, argv=argv@entry=0x55b80edb0700, dbname=0x55b80edb05f8 "test_crash", username=<optimized out>) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/postgres.c:4128
        #25 0x000055b80ce1187e in BackendRun (port=0x55b80eda76a0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:4408
        #26 BackendStartup (port=0x55b80eda76a0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:4080
        #27 ServerLoop () at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:1756
        #28 0x000055b80d06b819 in PostmasterMain (argc=5, argv=<optimized out>) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:1364
        #29 0x000055b80ce12c75 in main (argc=5, argv=0x55b80ed5a850) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/main/main.c:228
        ```
    - generated from old core dump
        ```
        GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
        Copyright (C) 2016 Free Software Foundation, Inc.
        License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
        This is free software: you are free to change and redistribute it.
        There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
        and "show warranty" for details.
        This GDB was configured as "x86_64-linux-gnu".
        Type "show configuration" for configuration details.
        For bug reporting instructions, please see:
        <http://www.gnu.org/software/gdb/bugs/>.
        Find the GDB manual and other documentation resources online at:
        <http://www.gnu.org/software/gdb/documentation/>.
        For help, type "help".
        Type "apropos word" to search for commands related to "word"...
        Reading symbols from /usr/lib/postgresql/10/bin/postgres...Reading symbols from /usr/lib/debug/.build-id/be/2d0e659760671fda0395f73e15ef668c2c4235.debug...done.
        done.
        [New LWP 29029]
        [Thread debugging using libthread_db enabled]
        Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
        Core was generated by `postgres: 10/test: postgres test_crash [local] CREATE INDEX                   '.
        Program terminated with signal SIGSEGV, Segmentation fault.
        #0  pfree (pointer=0x7f7a6e1648f8) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/mmgr/mcxt.c:954
        954	/build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/mmgr/mcxt.c: Nie ma takiego pliku ani katalogu.
        (gdb) bt

        #0  pfree (pointer=0x7f7a6e1648f8) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/mmgr/mcxt.c:954
        #1  0x00007f7a65e69b5e in ?? () from /usr/lib/postgresql/10/lib/postgis-2.5.so
        #2  0x00007f7a65e6b367 in gserialized_gist_picksplit () from /usr/lib/postgresql/10/lib/postgis-2.5.so
        #3  0x000055b80d1f6272 in FunctionCall2Coll (flinfo=flinfo@entry=0x55b80ee8f568, collation=<optimized out>, arg1=arg1@entry=94249012656136, arg2=arg2@entry=140736435822256) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/utils/fmgr/fmgr.c:1059
        #4  0x000055b80ce3b526 in gistUserPicksplit (len=227, giststate=0x55b80ee8d748, itup=0x55b80eea2fd8, v=0x7fffc1439eb0, attno=0, entryvec=0x55b80eeb7808, r=0x7f7a660d10a8)
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistsplit.c:433
        #5  gistSplitByKey (r=r@entry=0x7f7a660d10a8, page=page@entry=0x7f7a6e162a00 "", itup=itup@entry=0x55b80eea2fd8, len=len@entry=227, giststate=giststate@entry=0x55b80ee8d748, v=v@entry=0x7fffc1439eb0, attno=0)
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistsplit.c:697
        #6  0x000055b80ce3252c in gistSplit (r=r@entry=0x7f7a660d10a8, page=page@entry=0x7f7a6e162a00 "", itup=0x55b80eea2fd8, len=227, giststate=giststate@entry=0x55b80ee8d748) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:1375
        #7  0x000055b80ce32952 in gistplacetopage (rel=0x7f7a660d10a8, freespace=<optimized out>, giststate=giststate@entry=0x55b80ee8d748, buffer=183, itup=itup@entry=0x7fffc143a388, ntup=ntup@entry=1, oldoffnum=0, newblkno=0x0, leftchildbuf=0, splitinfo=0x7fffc143a2c0, 
            markfollowright=1 '\001', heapRel=0x7f7a660cf9b8) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:295
        #8  0x000055b80ce3341e in gistinserttuples (state=state@entry=0x7fffc143a390, stack=stack@entry=0x7fffc143a3b0, giststate=giststate@entry=0x55b80ee8d748, tuples=tuples@entry=0x7fffc143a388, ntup=ntup@entry=1, oldoffnum=oldoffnum@entry=0, leftchild=0, rightchild=0, 
            unlockbuf=0 '\000', unlockleftchild=0 '\000') at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:1221
        #9  0x000055b80ce3382a in gistinserttuple (oldoffnum=0, tuple=0x55b80eea2fa8, giststate=0x55b80ee8d748, stack=0x7fffc143a3b0, state=0x7fffc143a390) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:1180
        #10 gistdoinsert (r=r@entry=0x7f7a660d10a8, itup=itup@entry=0x55b80eea2fa8, freespace=<optimized out>, giststate=0x55b80ee8d748, heapRel=<optimized out>) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gist.c:837
        #11 0x000055b80ce3cd11 in gistBuildCallback (index=index@entry=0x7f7a660d10a8, htup=htup@entry=0x55b80edad700, values=values@entry=0x7fffc143a590, isnull=isnull@entry=0x7fffc143a8e0 "", tupleIsAlive=tupleIsAlive@entry=1 '\001', state=state@entry=0x7fffc143a9a0)
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistbuild.c:488
        #12 0x000055b80ceb15e7 in IndexBuildHeapRangeScan (heapRelation=heapRelation@entry=0x7f7a660cf9b8, indexRelation=indexRelation@entry=0x7f7a660d10a8, indexInfo=indexInfo@entry=0x55b80eda6428, allow_sync=allow_sync@entry=1 '\001', anyvisible=anyvisible@entry=0 '\000', 
            start_blockno=start_blockno@entry=0, numblocks=4294967295, callback=0x55b80ce3cc80 <gistBuildCallback>, callback_state=0x7fffc143a9a0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:2701
        #13 0x000055b80ceb1b0c in IndexBuildHeapScan (heapRelation=heapRelation@entry=0x7f7a660cf9b8, indexRelation=indexRelation@entry=0x7f7a660d10a8, indexInfo=indexInfo@entry=0x55b80eda6428, allow_sync=allow_sync@entry=1 '\001', 
            callback=callback@entry=0x55b80ce3cc80 <gistBuildCallback>, callback_state=callback_state@entry=0x7fffc143a9a0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:2271
        #14 0x000055b80ce3d380 in gistbuild (heap=0x7f7a660cf9b8, index=0x7f7a660d10a8, indexInfo=0x55b80eda6428) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/access/gist/gistbuild.c:207
        #15 0x000055b80ceb2923 in index_build (heapRelation=heapRelation@entry=0x7f7a660cf9b8, indexRelation=indexRelation@entry=0x7f7a660d10a8, indexInfo=indexInfo@entry=0x55b80eda6428, isprimary=isprimary@entry=0 '\000', isreindex=isreindex@entry=0 '\000')
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:2132
        #16 0x000055b80ceb42b9 in index_create (heapRelation=heapRelation@entry=0x7f7a660cf9b8, indexRelationName=indexRelationName@entry=0x55b80eda6e78 "test_table_geom_idx", indexRelationId=540674, indexRelationId@entry=0, relFileNode=<optimized out>, 
            indexInfo=indexInfo@entry=0x55b80eda6428, indexColNames=indexColNames@entry=0x55b80edadb08, accessMethodObjectId=783, tableSpaceId=1663, collationObjectId=0x55b80edadb50, classObjectId=0x55b80edadb68, coloptions=0x55b80edadb80, reloptions=0, isprimary=0 '\000', 
            isconstraint=0 '\000', deferrable=0 '\000', initdeferred=0 '\000', allow_system_table_mods=0 '\000', skip_build=0 '\000', concurrent=0 '\000', is_internal=0 '\000', if_not_exists=0 '\000')
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/catalog/index.c:1133
        #17 0x000055b80cf518b0 in DefineIndex (relationId=<optimized out>, relationId@entry=534024, stmt=stmt@entry=0x55b80eda6ea8, indexRelationId=indexRelationId@entry=0, is_alter_table=is_alter_table@entry=0 '\000', check_rights=check_rights@entry=1 '\001', 
            check_not_in_use=check_not_in_use@entry=1 '\001', skip_build=0 '\000', quiet=0 '\000') at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/commands/indexcmds.c:682
        #18 0x000055b80d0e2f80 in ProcessUtilitySlow (pstate=pstate@entry=0x55b80eda62c8, pstmt=pstmt@entry=0x55b80ee33120, 
            queryString=queryString@entry=0x55b80ee31de8 "drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;", context=context@entry=PROCESS_UTILITY_QUERY, 
            params=params@entry=0x0, queryEnv=queryEnv@entry=0x0, completionTag=0x7fffc143b3f0 "", dest=0x55b80ee33360) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/utility.c:1336
        #19 0x000055b80d0e1f87 in standard_ProcessUtility (pstmt=0x55b80ee33120, 
            queryString=0x55b80ee31de8 "drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;", context=PROCESS_UTILITY_QUERY, params=0x0, queryEnv=0x0, 
            dest=0x55b80ee33360, completionTag=0x7fffc143b3f0 "") at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/utility.c:931
        #20 0x000055b80d0defab in PortalRunUtility (portal=0x55b80ed79d48, pstmt=0x55b80ee33120, isTopLevel=<optimized out>, setHoldSnapshot=<optimized out>, dest=<optimized out>, completionTag=0x7fffc143b3f0 "")
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/pquery.c:1178
        #21 0x000055b80d0dfa98 in PortalRunMulti (portal=portal@entry=0x55b80ed79d48, isTopLevel=isTopLevel@entry=0 '\000', setHoldSnapshot=setHoldSnapshot@entry=0 '\000', dest=dest@entry=0x55b80ee33360, altdest=altdest@entry=0x55b80ee33360, 
            completionTag=completionTag@entry=0x7fffc143b3f0 "") at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/pquery.c:1331
        #22 0x000055b80d0e0805 in PortalRun (portal=portal@entry=0x55b80ed79d48, count=count@entry=9223372036854775807, isTopLevel=isTopLevel@entry=0 '\000', run_once=run_once@entry=1 '\001', dest=dest@entry=0x55b80ee33360, altdest=altdest@entry=0x55b80ee33360, 
            completionTag=0x7fffc143b3f0 "") at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/pquery.c:799
        #23 0x000055b80d0dc2bc in exec_simple_query (query_string=0x55b80ee31de8 "drop index if exists test_table_geom_idx; CREATE INDEX test_table_geom_idx ON public.test_table USING gist(st_intersection gist_geometry_ops_nd) TABLESPACE pg_default;")
            at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/postgres.c:1122
        #24 0x000055b80d0dd7e8 in PostgresMain (argc=<optimized out>, argv=argv@entry=0x55b80eda96b0, dbname=0x55b80eda95a8 "test_crash", username=<optimized out>) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/tcop/postgres.c:4128
        #25 0x000055b80ce1187e in BackendRun (port=0x55b80eda2de0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:4408
        #26 BackendStartup (port=0x55b80eda2de0) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:4080
        #27 ServerLoop () at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:1756
        #28 0x000055b80d06b819 in PostmasterMain (argc=5, argv=<optimized out>) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/postmaster/postmaster.c:1364
        #29 0x000055b80ce12c75 in main (argc=5, argv=0x55b80ed5a850) at /build/postgresql-10-0Kn02a/postgresql-10-10.12/build/../src/backend/main/main.c:228

        ```


    OLDEST GDB:
    ```
    GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
    Copyright (C) 2016 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
    and "show warranty" for details.
    This GDB was configured as "x86_64-linux-gnu".
    Type "show configuration" for configuration details.
    For bug reporting instructions, please see:
    <http://www.gnu.org/software/gdb/bugs/>.
    Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.
    For help, type "help".
    Type "apropos word" to search for commands related to "word"...
    Reading symbols from /usr/lib/postgresql/10/bin/postgres...(no debugging symbols found)...done.
    [New LWP 29029]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
    Core was generated by `postgres: 10/test: postgres test_crash [local] CREATE INDEX                   '.
    Program terminated with signal SIGSEGV, Segmentation fault.
    #0  0x000055b80d21845b in pfree ()
    (gdb) bt full
    #0  0x000055b80d21845b in pfree ()
    No symbol table info available.
    #1  0x00007f7a65e69b5e in ?? () from /usr/lib/postgresql/10/lib/postgis-2.5.so
    No symbol table info available.
    #2  0x00007f7a65e6b367 in gserialized_gist_picksplit () from /usr/lib/postgresql/10/lib/postgis-2.5.so
    No symbol table info available.
    #3  0x000055b80d1f6272 in FunctionCall2Coll ()
    No symbol table info available.
    #4  0x000055b80ce3b526 in gistSplitByKey ()
    No symbol table info available.
    #5  0x000055b80ce3252c in gistSplit ()
    No symbol table info available.
    #6  0x000055b80ce32952 in gistplacetopage ()
    No symbol table info available.
    #7  0x000055b80ce3341e in ?? ()
    No symbol table info available.
    #8  0x000055b80ce3382a in gistdoinsert ()
    No symbol table info available.
    #9  0x000055b80ce3cd11 in ?? ()
    No symbol table info available.
    #10 0x000055b80ceb15e7 in IndexBuildHeapRangeScan ()
    No symbol table info available.
    #11 0x000055b80ceb1b0c in IndexBuildHeapScan ()
    No symbol table info available.
    #12 0x000055b80ce3d380 in gistbuild ()
    No symbol table info available.
    #13 0x000055b80ceb2923 in index_build ()
    No symbol table info available.
    #14 0x000055b80ceb42b9 in index_create ()
    No symbol table info available.
    #15 0x000055b80cf518b0 in DefineIndex ()
    No symbol table info available.
    #16 0x000055b80d0e2f80 in ?? ()
    No symbol table info available.
    #17 0x000055b80d0e1f87 in standard_ProcessUtility ()
    No symbol table info available.
    #18 0x000055b80d0defab in ?? ()
    No symbol table info available.
    #19 0x000055b80d0dfa98 in ?? ()
    No symbol table info available.
    #20 0x000055b80d0e0805 in PortalRun ()
    No symbol table info available.
    #21 0x000055b80d0dc2bc in ?? ()
    No symbol table info available.
    #22 0x000055b80d0dd7e8 in PostgresMain ()
    No symbol table info available.
    #23 0x000055b80ce1187e in ?? ()
    No symbol table info available.
    #24 0x000055b80d06b819 in PostmasterMain ()
    No symbol table info available.
    #25 0x000055b80ce12c75 in main ()
    No symbol table info available.
    (gdb) 
    ```



~I don't know how to restore errors below but on other db I had system crash file.~
_Abowe is CoreDump from this example - now system was generate this file_

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
