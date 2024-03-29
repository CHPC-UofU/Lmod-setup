# CHPCs Lmod 
* [Modules setup](#modulesetup)
    * [System spider cache](#scache)

* [Module file format](#moduleformat)
    * [Version variable](#version)
    * [Help section](#help)
    * [Description section](#whatis)
    * [Environment variables](#env)
* [Other module file format features](#moduleformatother)
    * [Dependencies](#depend)
    * [Module properties](#prop)
    * [Family](#family)
    * [Defining aliases in the module files](#alias)
    * [Restricting module use to certain groups](#lic)
    * [Module hiding, versions and aliases](#ver)
    * [Modules for complicated initialization scripts](#shinit)
        * [Autogenerate lua module file](#autog)
        * [Source init script in the module file and unset the variables at module unload](#source)
    * [Python and Anaconda Virtual Environments](#pve)
* [Usage monitoring](#usage)
    * [Setup](#usagesetup)
    * [Using SQL database](#sql)
    * [Using Elasticsearch](#elk)

## <a name="modulesetup"></a>Modules setup

We have a specific installation strategy for Lmod in order to have different versions to co-exist in the sys branch. We follow the installation instructions at [http://lmod.readthedocs.io/en/latest/030_installing.html](http://lmod.readthedocs.io/en/latest/030_installing.html), but, with a twist.

* Download the Lmod source to the srcdir
* make the cache directories, e.g.
```
$ mkdir -p /uufs/chpc.utah.edu/sys/installdir/lmod/systemfiles/7.7-c7/
$ touch /uufs/chpc.utah.edu/sys/installdir/lmod/systemfiles/7.7-c7/system.txt
$ mkdir -p /uufs/chpc.utah.edu/sys/installdir/lmod/cache/7.7-c7
```
* In srcdir, configure and preinstall:
```
$ ./configure --prefix=/uufs/chpc.utah.edu/sys/installdir --with-module-root-path=/uufs/chpc.utah.edu/sys/modulefiles/CHPC-c7 --with-spiderCacheDir=/uufs/chpc.utah.edu/sys/installdir/lmod/cache/7.7-c7 --with-updateSystemFn=/uufs/chpc.utah.edu/sys/installdir/lmod/systemfiles/7.7-c7/system.txt --with-colorize=YES --with-tcl=YES --with-autoSwap=YES --with-useDotFiles=YES --with-mpathSearch=yes
$ make pre-install
```
* Modify the installed files to replace `lmod/lmod` references. This allows us to run lmod directly from the given version directory (in this case 7.7.29):
```
$ grep -rl "lmod\/lmod" * | xargs sed -i 's/lmod\/lmod/lmod\/7.7.29/g'
```
* Modify `libexec/SitePackage.lua` - use the file from older version to add hooks for module load logging and for licensed programs checks
* Modify `init/lmodrc.lua` - to add the `host` module property - this is used for programs built both for GPU and CPU (host), e.g. vasp/5.4.4
* Test the new Lmod installation - this requires unloading the existing Lmod and starting the new one in an user shell. We have examples of scripts that do thia, e.g. `/uufs/chpc.utah.edu/sys/modulefiles/scripts/switch_to_18.csh`. Dont forget to source the file so that changes take effect.
* When ready to deploy, modify `/uufs/chpc.utah.edu/sys/etc/profile.d/module.[csh,sh]` to change the Lmod version to source

### <a name="scache"></a>System spider cache

We are using cache, which is being put to `/uufs/chpc.utah.edu/sys/installdir/lmod/cache`, different location for different Lmod version (or, better to say, different locations of our module files). More info on this is at [http://lmod.readthedocs.io/en/latest/130_spider_cache.html](http://lmod.readthedocs.io/en/latest/130_spider_cache.html).

Our particular setup involves running [`/uufs/chpc.utah.edu/sys/modulefiles/scripts/caching/update_cache.sh`](update_cache.sh). We run a cron job, currently on `centos7.chpc.utah.edu`, as hpcapps, every 10 minutes to refresh the spider cache by:
```
$ crontab -e
# Every 10 minutes: regenerate lmod module cache
1,11,21,31,41,51 * * * * /uufs/chpc.utah.edu/common/home/hpcapps/update_cache.sh >/dev/null 2>&1
```

One thing that we missed was that cron runs in a bare environment, which resulted in Lmod not being sourced and the spider cache being incomplete. This was solved by adding the following to the script above:
```
if [ -z "$LMOD_VERSION" ]; then
  source /etc/profile.d/chpc.sh
fi
```

We are also running a monitoring script to check on updates of the spider cache files to make sure the cron update is running. If the cache files are older than 13.3 hours, we send alert to operations. The file that is being checked is `/uufs/chpc.utah.edu/sys/installdir/lmod/cache/std/spiderT.lua`.

To run module commands without using the cache:
```
$ module --ignore_cache avail
```

### Environment variables 

Lets consider to change the following [environment variables](http://lmod.readthedocs.io/en/latest/090_configuring_lmod.html), which modify Lmods behavior:
```
LMOD_PIN_VERSIONS - default - no - may want to set to yes to module restore the same versions of modules which were used with module save.
LMOD_SHORTTIME - default - 2 - set to large value (86400) to prevent user spider cache to be generated - this has been turned on since automatic spider cache generation is active.
```

## <a name="moduleformat"></a>Module file format

We should strive to keep the module format somewhat consistent. Each module file should contain the following:

### <a name="version"></a>Version variable

It is advantageous to define the package version first since the module file can then refer to this throughout the module file, e.g.
```
local version = "18.1"
local base = pathJoin("/uufs/chpc.utah.edu/sys/installdir/pgi", version)
```

### <a name="help"></a>Help section

We need to agree on a format. I feel that some help pages are too long, e.g.
```
help(
[[
This module sets the following env. variables:
     a. PATH
     b. LD_LIBRARY_PATH
     c. the MANPATH variable
        You can invoke the man page as follows:
            man 3 netcdf

     d. The following env. variables:
        NETCDFC        :: /uufs/chpc.utah.edu/sys/installdir/netcdf-c/4.4.1-c7/
        NETCDFC_INCDIR :: /uufs/chpc.utah.edu/sys/installdir/netcdf-c/4.4.1-c7/include
        NETCDFC_LIBDIR :: /uufs/chpc.utah.edu/sys/installdir/netcdf-c/4.4.1-c7/lib

     e. This version of netcdf-c has been built with the FOLLOWING libraries:
        1./uufs/chpc.utah.edu/sys/installdir/zlib/1.2.8-c7/
        2./uufs/chpc.utah.edu/sys/installdir/szip/2.1-c7/
        3./uufs/chpc.utah.edu/sys/installdir/hdf5/1.8.17-c7/

for the netCDF-c package (v.4.4.1) for Centos7
]])

```

### <a name="whatis"></a>Description section

Description defines some tags associated with the module, which can be used in searching, e.g.:
```
whatis("Name         : PGI Compilers")
whatis("Version      : " .. version)
whatis("Category     : compiler")
whatis("Description  : Compiler (C, C++, Fortran)")
whatis("Keywords     : System, compiler")
whatis("URL          : http://www.pgroup.com/")
whatis("Installed on : 2/26/2018")
whatis("Installed by : Your Name")
```
Note that we are using the `version` variable defined at the start.

For Keywords, lets use the Tags used in the Application database, which are listed in [Tags.csv](Tags.csv). The application database categories, defined in [Categories.csv](Categories.csv), are not very detailed, but, it may not be a bad idea to start using that so we can potentially in the future group the modules based on these categories. We may expand the Applications, though, for different fields of science.

### <a name="env"></a>Environmental variables

We usually need to define PATH and other variables, and, when practical we should also define an environmental variable that specifies the package location and include/library location, e.g.
```
prepend_path("PATH",pathJoin(base, "/bin"))
prepend_path("LD_LIBRARY_PATH",pathJoin(base,"/lib"))
prepend_path("MANPATH",pathJoin(base,"/share/man/man3"))
prepend_path("PKG_CONFIG_PATH", pathJoin(base,"lib/pkgconfig"))
setenv("NETCDFC",base)
setenv("NETCDFC_INCDIR",pathJoin(base,"/include"))
setenv("NETCDFC_LIBDIR",pathJoin(base,"/lib"))
```
If the package is a library, we should add the LD_LIBRARY_PATH and possibly also PKG_CONFIG_PATH, if the package contains ```pkgconfig``` information.

## <a name="moduleformatother"></a>Other module file format features

All Lua module file functions are listed at [http://lmod.readthedocs.io/en/latest/050_lua_modulefiles.html](http://lmod.readthedocs.io/en/latest/050_lua_modulefiles.html).

### <a name="depend"></a>Dependencies

Dependencies can be included in several different ways. See [http://lmod.readthedocs.io/en/latest/098_dependent_modules.html](http://lmod.readthedocs.io/en/latest/098_dependent_modules.html) for details. Most common possibilities are summarized below.

#### <a name="hier"></a>Hierarchy

We handle direct dependencies on compilers, MPI, CUDA and potentially other packages (Python, Anaconda, R) via hierarchy. Compilers and MPI should be always done this way, for other tools only if they dont depend on a compiler and MPI (e.g. if we have a MPI parallel CUDA code, the MPI dependency will be handled by the hierarchy and CUDA dependency explicitly in one of the way below).

#### <a name=""></a>Use of RPATH

In general the best approach for library dependencies is hardcoding the dynamic library path in the executable via RPATH, i.e. linking as
```
-Lmy_lib_path -lmy_lib -Wl,-rpath=my_lib_path
```

#### <a name=""></a>Explicit loading

For dependencies that require other things than dynamic libraries (e.g. executables from bin directory), the best option for explicit loading is `depends_on()`. Using that will load the dependent module if its not loaded yet, unload when the original module is unloaded, but keep the dependent module if it has been loaded earlier.
```
depends_on("cuda/9.1")
```
#### <a name=""></a>Prerequisite definition

For expert users, we may use the `prereq()` function. If the dependent module is not loaded, loading a module with `prereq()` will error out with a message that the prerequisite module has not been loaded. This leaves the dependency handling on the user.

### <a name="prop"></a>Module properties via labels

Modules can be labelled for different groupings. [http://lmod.readthedocs.io/en/latest/145_properties.html#lmodrc-label](http://lmod.readthedocs.io/en/latest/145_properties.html#lmodrc-label) There are two default labels:

#### <a name=""></a>State

`experimental`,`testing`,`obsolete`. 

I propose to mark older version obsolete, AND potentially hide them (list them as hidden in `/uufs/chpc.utah.edu/sys/modulefiles/etc/rc`):
```
add_property("state","obsolete")
```

#### <a name=""></a>Architecture

`gpu`,`mic`,...

For our purposes we should mark GPU built packages with this label, e.g.
```
add_property("arch","gpu")
```
I have also defined a new property, `host`, which should be used for packages which have been built both for GPU and CPU (host):
```
add_property("arch","gpu:host")
```


### <a name="family"></a>Family

Defines that only one module in a family can be loaded at a time, e.g.

```
family("R")
```
List of families:
* Compiler 
* mpi
* R - own built R or OpenR
* Python - own built, Anaconda, Intel Python
* hdf5 - hdf5, phdf5
* boost - boost, pboost
* java - Oracle or OpenJDK
* matlab - in case package uses old Matlab for which we dont have modules and have hard coded paths (delft3dvis)
* idl - potential similar need to matlab above
* gaussian 

Questionable families:
libflame, scala, cuda, julia, spark, gromacs, hoomd

### <a name="alias"></a>Defining aliases in the module files

Command aliases are useful, and Lmod defines `set_alias()` function for that purpose. However, aliases dont get expanded in bash non-interactive (e.g. job scripts) shells. Therefore, instead of using `set_alias()` function in the module files, we should use shell functions using the `set_shell_function()` function. Furthermore, in Bash, we need to `export` the newly created shell function. Therefore, the whole alias creation of `newcmd` pointing to `oldcmd` is as follows:
```
local bashStr = 'orgcmd "$@"'
local cshStr  = "orgcmd $*"
set_shell_function("newcmd",bashStr,cshStr)
if (myShellName() == "bash") then
 execute{cmd="export -f newcmd",modeA={"load"}}
end
```
The `execute` function runs a given command `cmd` in a given Lmod mode `modeA` - in this case it will export the newcmd function when the module is loaded.

**NOTE - aliases dont work with `mpirun`**. `mpirun` under the hood calls a binary program launcher, which does not expand the aliases, or shell functions, e.g. in bash:
```
$ myfunction(){ ./a.out; }
$ export -f myfunction
$ mpirun -np 1 myfunction
 [proxy:0:0@notchpeak1] HYDU_create_process (../../utils/launch/launch.c:825): execvp error on file myfunction (No such file or directory)
```
The mpirun sh script calls `+ mpiexec.hydra -np 1 myfunction` and mpiexec.hydra somehow calls program in the argument (here `myfunction`). Since the system() call expands the exported shell function correctly, mpiexec.hydra probably calls ssh or similar to launch a new remote shell. The alias is not known in the remote shell and the launch fails.


### <a name="lic"></a>Restricting module use to certain groups

Licensing resctriction to certain groups or using different license information for different groups is done using the ```required_group``` hook from ```libexec/SitePackages.lua```. An example of its usage is below:
```
local err_message="To use this module you must be in a particular group\n" ..
                  "Please contact issues@chpc.utah.edu to find out more\n"

local found = required_group("starccmsub2") or required_group("starccmsub1") or required_group("starccm")

if (not found) then
   LmodError(err_message)
end

-- setup for USU licensing
local usu = required_group("usu")
if (usu) then
  setenv("CDLMD_LICENSE_FILE","17020@flexnet.it.usu.edu")
--  setenv("CDLMD_LICENSE_FILE","17000@elicense.eng.usu.edu")
else
  setenv("CDLMD_LICENSE_FILE","/uufs/chpc.utah.edu/sys/installdir/star-ccm+/license3_17.dat")
end
```

### <a name="ver"></a>Module hiding, versions and aliases

We can create shorter version or an alias for a module by a definition in `/uufs/chpc.utah.edu/sys/modulefiles/etc/rc`.  This is recommended for modules with long versions or where versions differ significantly, e.g.:
```
 module-version intel/2018.1.163 18.1 18
 module-version intel/2018.0.128 18.0 
 module-version lumerical/8.19.1466 8.19 2018a 18a
 module-alias python2 python/2.7.11
 module-alias python3 python/3.5.2
```

We can also hide older modules in the same rc file with e.g.:
```
 hide-version R/3.2.1
```

We should make a habit to hide older modules as we install newer versions of programs.

### <a name="shinit"></a>Modules for complicated initialization scripts

Some programs come with complicated environment setup scripts, which are not easy to transcribe to Lmods lua format. There are several ways how to deal with this problem.

#### <a name="autog"></a>Autogenerate lua module file

Lmod comes with a script [sh_to_modulefile](https://lmod.readthedocs.io/en/latest/260_sh_to_modulefile.html), which converts environment variables set in shell init scripts into lua format. We have historically been using [something similar](env). This works as long as the shell init scripts dont set aliases or shell functions.

#### <a name="source"></a>Source init script in the module file and unset the variables at module unload

If the init script sets aliases and/or shell functions, there are two ways to deal with that. One of them is to encode all the aliases and shell functions in the module file, which can get quite complicated. We feel that an easier way is to call the init script from the module file to initialize the environment, and then unset all the aliases and shell function during module unload.

To find out what is being set, for environment variables run `env` before and after the init script sourcing, save into a file and `gvim -d` them to see the difference. For aliases, do the same with the `alias` command. For shell function, use `declare -F` in bash. 

Use `unsetenv` or `unset` for environment variables in tcsh and bash, `unalias` to unset aliases and `unset -f` to unset a bash shell function.

For example, in the [anaconda3](2019.03.lua) module, to source the environment, we simply
```
execute{cmd="source " .. base .. "/etc/profile.d/conda."..myShellType(),modeA={"load"}}
```
Upon unload, we need to unset everything that has been sourced, which happens to be different for csh and sh shells, so, we need to do all of the following:
```
if (myShellType() == "csh") then
  -- csh sets these environment variables and an alias for conda
  cmd = "unsetenv CONDA_EXE; unsetenv _CONDA_ROOT; unsetenv _CONDA_EXE; " ..
        "unsetenv CONDA_SHLVL; unalias conda"
  execute{cmd=cmd, modeA={"unload"}}
end
if (myShellType() == "sh") then
  -- bash sets environment variables, shell functions and path to condabin
  if (mode() == "unload") then
    remove_path("PATH", pathJoin(base,"condabin"))
  end
  cmd = "unset CONDA_EXE; unset _CE_CONDA; unset _CE_M; " ..
        "unset -f __conda_activate; unset -f __conda_reactivate; " ..
        "unset -f __conda_hashr; unset CONDA_SHLVL; unset _CONDA_EXE; " ..
        "unset _CONDA_ROOT; unset -f conda"
  execute{cmd=cmd, modeA={"unload"}}
end
```
### <a name="pve"></a>Python and Anaconda Virtual Environments

It is possible and recommended to have modules for programs that live inside of virtual environments to activate and deactivate the VE upon module load and unload. For example, for the QIIME2 package, we can simply do:
```
   execute{cmd="conda activate /uufs/chpc.utah.edu/sys/installdir/qiime2/2019.4",modeA={"load"}}
   execute{cmd="conda deactivate",modeA={"unload"}}
```
Notice that this virtual environment was created at the application specific path, e.g. `conda env create --file qiime2-2019.4-py36-linux-conda.yml --prefix /uufs/chpc.utah.edu/sys/installdir/qiime2/2019.4` and then activates as `/uufs/chpc.utah.edu/sys/installdir/qiime2/2019.4`. The advantage of this approach is a separation of the Anaconda and the VE installations, disadvantage is that the name of the VE is the long path which shows in the terminal prompt after the VE activation.

Anaconda version specific installation is controlled via the modules hierarchy, e.g. this qiime2 module is at `/uufs/chpc.utah.edu/sys/modulefiles/CHPC-18/Compiler/anaconda/2019.03/qiime2`.

## <a name="usage"></a>Usage monitoring

### <a name="usagesetup"></a>Setup

We are roughly following instructions at [http://lmod.readthedocs.io/en/latest/300_tracking_module_usage.html](http://lmod.readthedocs.io/en/latest/300_tracking_module_usage.html).

* Modify SitePackage.lua to send a message to syslog whenever module load is invoked
  This sends a log entry to CHPCs syslog server

The syslog data storage and harvesting can be done with two methods

### <a name="sql"></a>Using SQL database

This is how Lmod author has it [described](http://lmod.readthedocs.io/en/latest/300_tracking_module_usage.html).

Note that the database search will be slower the more data will be put in the database, so, limiting the time in the search may be a good idea.

#### Setup

* Create and update an SQL database named `modlog` on `mysql.chpc.utah.edu`
* On the syslog server, run a daily cron job that `/uufs/chpc.utah.edu/sys/srcdir/lmod/7.7.29/contrib/tracking_module_usage/databaseCron.sh` harvests the logs and updates the `modlog` database calling `/uufs/chpc.utah.edu/sys/srcdir/lmod/7.7.29/contrib/tracking_module_usage/store_module_data`

#### Usage monitoring

##### Setup as of early 2024

Data from above get ingested into a database that's located in `/uufs/chpc.utah.edu/common/home/chpc-data/module-tracking`. One can use SQL commands to query the database, as:
```
cd /uufs/chpc.utah.edu/common/home/chpc-data/module-tracking
sqlite3 module-usage-tracking.db
```
Then run `.header on` to enable headers on output tables.

Example queries:
### Analyze usage of different versions of the same module (modify date range as desired)
```sql
select module, version, sum(count) from module_load where date between '2024-01-01' and '2025-01-01' and module='alphafold' group by module, version order by version desc;
```

### Look at details of recent module loads (who, where, when, what)
```sql
select * from module_load where date>='2024-01-01' and module='alphafold';
```

##### Older setup

The scripts below may still work or may need a minimal change to get working.

Lmod comes with a Python script `/uufs/chpc.utah.edu/sys/srcdir/lmod/7.7.29/contrib/tracking_module_usage/analyzeLmodDB` which queries the database for three scenarios [described here](http://lmod.readthedocs.io/en/latest/300_tracking_module_usage.html#step-7). We have made minor modifications to the script options to suit it to our needs.

* To list counts per host
```
$ /uufs/chpc.utah.edu/sys/srcdir/lmod/7.7.29/contrib/tracking_module_usage/analyzeLmodDB --sqlPattern '%plink%' counts

Module path                                                        Syshost        Distinct Users
-----------                                                        -------        --------------
/uufs/chpc.utah.edu/sys/modulefiles/CHPC-18/Core/plink/1.09.lua    kingspeak24                 1
/uufs/chpc.utah.edu/sys/modulefiles/CHPC-18/Core/plink/1.09.lua    kp240                       1
...
```
It may be more useful to aggregate the counts also over the hosts, and search for the module name, not module path (though module paths may be different for hierarchical modules like compilers and MPIs):
```
$ ./analyzeLmodDB --sqlPattern '%plink%' counts --all --name

Module name    Distinct Users
-----------    --------------
plink/1.09     2

```

* To list unique users of the module
```
$ /uufs/chpc.utah.edu/sys/srcdir/lmod/7.7.29/contrib/tracking_module_usage/analyzeLmodDB --sqlPattern '%plink%' usernames

Module path                                                        User Name    Syshost count
-----------                                                        -------      ---------
/uufs/chpc.utah.edu/sys/modulefiles/CHPC-18/Core/plink/1.09.lua    u0430776     4
/uufs/chpc.utah.edu/sys/modulefiles/CHPC-18/Core/plink/1.09.lua    u0806040     9
```

* To list modules used by a particular user
```
$ /uufs/chpc.utah.edu/sys/srcdir/lmod/7.7.29/contrib/tracking_module_usage/analyzeLmodDB --sqlPattern 'u0430776' modules_used_by

Module path                                                              Syshost      User Name
-----------                                                              -------      ---------
/uufs/chpc.utah.edu/sys/modulefiles/CHPC-18/Core/intel/2018.1.163.lua    lonepeak7    u0430776
/uufs/chpc.utah.edu/sys/modulefiles/CHPC-18/Core/plink/1.09.lua          lonepeak7    u0430776

```

* Other parameters
    * Limit hosts - `--syshost '%kingspeak%'`
    * Limit dates - `--start '2018-05-08'` - from 2018-05-08 till now
                  - `--start '2018-05-08'` `--end '2018-05-10'`
    * Aggregate over all hosts - `--all`
    * Search for module name, not module path - `--name`

### <a name="elk"></a>Using Elasticsearch/Kibana

This is still work in progress as we (Martin and Luan) are trying to figure out an efficient way to store and query the syslog data.

* Open FastX connection to redwood.chpc.utah.edu (ssh -X does not work)
* Open Firefox browser and go to `lachesis.int.chpc.utah.edu`. This is the Kibana server which talks to Elasticsearch server, `kerrigan.int.chpc.utah.edu`
* In Kibana, hit "Discover" and then use filters to filter out the `ModuleUsageTracking` messages aloing with optional additional filter, e.g.
```
message:ModuleUsageTracking AND message:"module=plink"
```

