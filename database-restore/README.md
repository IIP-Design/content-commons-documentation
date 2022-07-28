# Content Commons Database Restore

## Problem Statement
To execute a restore on the server void of foreign key constraint errors, the system needs to temporarily disable triggers during the restore. This can be accomplished by using the `--disable-triggers` flag as part the `pg_restore` command.  However, this is currently not possible on the server as AWS does not allow the required 'Superuser' privilege to be set which would allow the use of the `--disable-triggers` flag. In lieu of executing with the `--disable-triggers` flag, two approaches can be considered:
1. execute a `pg_dump` with the `--disable-triggers` flag
2. manual re-order and restore of database tables 

## Execute a `pg_dump` with the `--disable-triggers` flag
**Steps**
1. Execute the dump command. Note that the resulting dump file is a `sql` file and not an archive
2. Execute the prisma deploy command. This will create the schema and tables
3. Execute the restore command. Restore uses the `psql` command and not `pg_restore`. The `psql` command is used because the resulting dump file is a plain `sql` file and not an archive.
```
pg_dump -U <user> -d <database> --disable-triggers --superuser=<username> -a -v > commons.sql
npx prisma migrate deploy
psql -U <user> -d <database> < commons.sql 
```

Executing a dump with the `--disable-triggers` flag will add `ALTER` statements to the resulting sql dump file that will disable/enable table triggers on a per table basis:
```
ALTER TABLE prisma_prod."Bureau" DISABLE TRIGGER ALL;

COPY prisma_prod."Bureau" (id, name, abbr, "knownAs", "principalOfficerTitle") FROM stdin;
ck9ips7rq008l099706c9pdjv	Office of the Chief of Protocol	S/CPR		Ambassador
ck9ips7rr008m0997mqbr62cb	Office of the Coordinator for Cyber Issues	S/CCI		Coordinator
...

ALTER TABLE prisma_prod."Bureau" ENABLE TRIGGER ALL;
```
Note the use of the `--superuser=<username>` flag with the `pg_dump` statement. This will add a session authorization statement to the resulting sql dump file:
```
SET SESSION AUTHORIZATION '<username>';
```
Note that the session authorization statement can also be added manually to the sql dump file. 

**Issues**

While this approach did appear to restore successfully, some system trigger errors were observed during the restore process:
```
ERROR: permission denied: "RI_ConstraintTrigger_a_369754" is a system trigger
```
While these system errors seem to not impact the restore, this process should be tested and validated on the server.

## Manual re-order and restore of database tables 
**Steps**
1. Execute a data-only dump in the tar archive format
2. Generate a complete table list, e.g. db.list file
3. Separate db.list into different list files to control restore order
4. Restore each list in proper sequence

```
pg_dump -U <user> -d <database> -a -v -Ft -n '"commons$dev"' -f commons.tar 
pg_restore -U <user> -d <database> -l <name>.tar > db.list

# separate into different lists #

pg_restore  -U <user> -d <database> -a -v -L db-1.list <name>.tar
pg_restore  -U <user> -d <database> -a -v -L db-2.list <name>.tar
pg_restore  -U <user> -d <database> -a -v -L db-3.list <name>.tar
pg_restore  -U <user> -d <database> -a -v -L db-4.list <name>.tar
```
Another solution could be to reorder the tables in place within the db.list file itself.  The updated db.list could then be restored in a single restore execution. This option has not yet been tested.

**List content**

[View sample list files](https://github.com/IIP-Design/content-commons-documentation/tree/master/database-restore/Sample%20lists)

db-1.list
```
;
; Archive created at 2022-05-10 11:39:12 UTC
;     dbname: <database>
;     TOC Entries: 80
;     Compression: 0
;     Dump Version: 1.14-0
;     Format: TAR
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 13.5 (Debian 13.5-1.pgdg110+1)
;     Dumped by pg_dump version: 13.5 (Debian 13.5-1.pgdg110+1)
;
;
; Selected TOC Entries:
;
3663; 0 17722 TABLE DATA <schema> Bureau prisma
3664; 0 17728 TABLE DATA <schema> Category prisma
3686; 0 17854 TABLE DATA <schema> Region prisma
3667; 0 17743 TABLE DATA <schema> Country prisma
3668; 0 17749 TABLE DATA <schema> Dimensions prisma
3681; 0 17824 TABLE DATA <schema> Mission prisma
3669; 0 17752 TABLE DATA <schema> DiplomaticPost prisma
3673; 0 17776 TABLE DATA <schema> DocumentType prisma
3679; 0 17812 TABLE DATA <schema> Language prisma
3692; 0 17887 TABLE DATA <schema> Team prisma
3695; 0 17905 TABLE DATA <schema> User prisma
3670; 0 17758 TABLE DATA <schema> Document prisma
3674; 0 17782 TABLE DATA <schema> DocumentUse prisma
3675; 0 17788 TABLE DATA <schema> GraphicProject prisma
3676; 0 17794 TABLE DATA <schema> GraphicStyle prisma
3678; 0 17806 TABLE DATA <schema> ImageUse prisma
3680; 0 17818 TABLE DATA <schema> LanguageTranslation prisma
3682; 0 17830 TABLE DATA <schema> Office prisma
3683; 0 17836 TABLE DATA <schema> Package prisma
3685; 0 17848 TABLE DATA <schema> PolicyPriority prisma
3684; 0 17842 TABLE DATA <schema> Playbook prisma
3687; 0 17860 TABLE DATA <schema> SocialPlatform prisma
3688; 0 17866 TABLE DATA <schema> SocialResource prisma
3690; 0 17878 TABLE DATA <schema> SupportFileUse prisma
3691; 0 17884 TABLE DATA <schema> Tag prisma
3694; 0 17899 TABLE DATA <schema> Toolkit prisma
3696; 0 17911 TABLE DATA <schema> UserNotification prisma
3701; 0 17944 TABLE DATA <schema> VideoUse prisma
3698; 0 17926 TABLE DATA <schema> VideoProject prisma
3700; 0 17938 TABLE DATA <schema> VideoUnit prisma
```

db-2.list
```
;
; Archive created at 2022-05-10 11:39:12 UTC
;     dbname: <database>
;     TOC Entries: 80
;     Compression: 0
;     Dump Version: 1.14-0
;     Format: TAR
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 13.5 (Debian 13.5-1.pgdg110+1)
;     Dumped by pg_dump version: 13.5 (Debian 13.5-1.pgdg110+1)
;
;
; Selected TOC Entries:
;
3665; 0 17731 TABLE DATA <schema> CommonsResource prisma
3666; 0 17737 TABLE DATA <schema> ContentField prisma
3672; 0 17770 TABLE DATA <schema> DocumentFile prisma 
3689; 0 17872 TABLE DATA <schema> SupportFile prisma
3697; 0 17920 TABLE DATA <schema> VideoFile prisma
3699; 0 17932 TABLE DATA <schema> VideoStream prisma
```

db-3.list
```
;
; Archive created at 2022-05-10 11:39:12 UTC
;     dbname: <database>
;     TOC Entries: 80
;     Compression: 0
;     Dump Version: 1.14-0
;     Format: TAR
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 13.5 (Debian 13.5-1.pgdg110+1)
;     Dumped by pg_dump version: 13.5 (Debian 13.5-1.pgdg110+1)
;
;
; Selected TOC Entries:
;
 
3671; 0 17764 TABLE DATA <schema> DocumentConversionFormat prisma
3677; 0 17800 TABLE DATA <schema> ImageFile prisma 
3693; 0 17896 TABLE DATA <schema> Thumbnail prisma
```

db-4.list
```
;
; Archive created at 2022-05-10 11:39:12 UTC
;     dbname: <database>
;     TOC Entries: 80
;     Compression: 0
;     Dump Version: 1.14-0
;     Format: TAR
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 13.5 (Debian 13.5-1.pgdg110+1)
;     Dumped by pg_dump version: 13.5 (Debian 13.5-1.pgdg110+1)
;
;
; Selected TOC Entries:
;
3702; 0 17950 TABLE DATA <schema> _BureauToDocument prisma
3703; 0 17953 TABLE DATA <schema> _BureauToDocumentFile prisma
3704; 0 17956 TABLE DATA <schema> _CategoryToDocument prisma
3705; 0 17959 TABLE DATA <schema> _CategoryToDocumentFile prisma
3706; 0 17962 TABLE DATA <schema> _CategoryToGraphicProject prisma
3707; 0 17965 TABLE DATA <schema> _CategoryToLanguageTranslation prisma
3708; 0 17968 TABLE DATA <schema> _CategoryToPackage prisma
3709; 0 17971 TABLE DATA <schema> _CategoryToPlaybook prisma
3710; 0 17974 TABLE DATA <schema> _CategoryToToolkit prisma
3711; 0 17977 TABLE DATA <schema> _CategoryToVideoProject prisma
3712; 0 17980 TABLE DATA <schema> _CategoryToVideoUnit prisma
3713; 0 17986 TABLE DATA <schema> _CountryToDocument prisma
3714; 0 17989 TABLE DATA <schema> _CountryToDocumentFile prisma
3715; 0 17992 TABLE DATA <schema> _CountryToMission prisma
3716; 0 17995 TABLE DATA <schema> _CountryToPlaybook prisma
3717; 0 17998 TABLE DATA <schema> _DiplomaticPostToDocument prisma
3718; 0 18001 TABLE DATA <schema> _DirectlyRelatedBureaus prisma
3719; 0 18007 TABLE DATA <schema> _DocumentFileToOffice prisma
3720; 0 18013 TABLE DATA <schema> _DocumentFileToRegion prisma
3721; 0 18016 TABLE DATA <schema> _DocumentFileToTag prisma
3722; 0 18022 TABLE DATA <schema> _DocumentToMission prisma
3723; 0 18025 TABLE DATA <schema> _DocumentToOffice prisma
3724; 0 18028 TABLE DATA <schema> _DocumentToPolicyPriority prisma
3725; 0 18031 TABLE DATA <schema> _DocumentToRegion prisma
3726; 0 18034 TABLE DATA <schema> _DocumentToTag prisma
3727; 0 18037 TABLE DATA <schema> _DocumentToTeam prisma
3728; 0 18046 TABLE DATA <schema> _GraphicProjectToTag prisma
3729; 0 18049 TABLE DATA <schema> _ImageFileToSocialPlatform prisma
3730; 0 18055 TABLE DATA <schema> _LanguageTranslationToTag prisma
3731; 0 18058 TABLE DATA <schema> _PackageToTag prisma
3732; 0 18061 TABLE DATA <schema> _PlaybookToRegion prisma
3733; 0 18070 TABLE DATA <schema> _PlaybookToTag prisma
3734; 0 18079 TABLE DATA <schema> _TagToToolkit prisma
3735; 0 18082 TABLE DATA <schema> _TagToVideoProject prisma
3736; 0 18085 TABLE DATA <schema> _TagToVideoUnit prisma
3737; 0 18091 TABLE DATA <schema> _UserToUserNotification prisma
```