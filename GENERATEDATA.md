# How to generate data files

## Notes
The scripts are relatively raw and will be refined over time.

To create the datafiles download the data generator from http://www.cs.brandeis.edu/~linearroad/tools.html.

### Using the original generator
To get the original generator working on CentOS6.5/6, or other modern 64-bit Linux distribution, the 32-bit compatibility pack must be installed.  If an older, 32-bit version of Linux is available (i.e. 32-bit CentOS 4.8) that works too.  Or, you could try recompiling the mitsim program into a 64-bit version. 

Both a 64-bit OS (CentOS in Azure) with the 32-bit compatibility pack installed and a 32-bit CentOS 4.8 install on a private machine were both successful.

The general steps for Centos 6.5/6 follow:

Download the original tools and unpack into arbitrary directory:

```
wget http://www.cs.brandeis.edu/~linearroad/files/mitsim.tar.gz
mkdir MITSIMLab
cd MITSIMLab
tar xf ../mitsim.tar.gz`
```

Install and set up the PostgreSQL database (these instructions may vary based on the version of PostgreSQL).  For version 8.4.0 that the default CentOS 6.5/6 repo in Azure installs:

```
sudo yum -y install postgresql postgresql-server
sudo service postgresql initdb
sudo service postgresql start
su postgres
psql
psql> create user <linux username>;  # this should be the same username from which scripts will be run
psql> alter role <linux username> with superuser login;
psql> create database test;
```

Install gcc and make if not already installed.
```
sudo yum -y install gcc make
```
Install the appropriate Perl modules for the scripts to interact with postgresql.
```
perl -MCPAN -e "install DBI"
perl -MCPAN -e "install DBD::PgPP"
perl -MCPAN -e "install Math::Random"
```
Install the 32-bit compatibility pack:
```
sudo yum install compat-libstdc++-296.i686
```
You should now have PostgreSQL setup with an appropriate user and database along with the proper Perl modules.  To test database connectivity modify the included *test.pl* file to point to the new database connection: 
```
DBI->connect("DBI:PgPP:dbname=test", "root", "")
```
and insert a `print $dbh;` statement after the connection statement to test for connectivity.  If something prints the connection should be good.

### Creating a single combined data file
As stated in the README, datasets of arbitrary sizes can be generated on a single machine or by parallelizing the expressway generation on multiple machines.  But, after generation, these must be cleaned (if desired) and combined.  

**These are the scripts and commands used for cleaning raw files--run on the individual raw files.  (Any number of additional steps can be added as desired.)**

```
dataval.py <raw_file>  <temp_outfile>
datarm2.py <temp_outfile> > <temp_outfile2>  # remove carids with only <=2 tuples
datamakeexit.py <temp_outfile2> > <temp_outfile3>  # make the last type 0 record an exit lane tuple
mv <temp_outfile3> <clean_file>
```
After cleaning, merge the _n_ "clean" files.
```
datacombine.py  <dir_of_cleaned_files>  <combined_cleaned_file>
```
Then, create the tolls and the random re-entrant cars.
```
combine.py <combined_cleaned_file> <output_dir> <num_xways>
  # combine.py uses: p_duplicates.py, historical-tolls.pl
  # Also, pre-create the following files in the <output_dir> and change permissions accordingly:
touch carsandtimes.csv; touch carstoreplace.csv; chmod 777 carsandtimes.csv; chmod 777 carstoreplace.csv 
  #These steps are necessary as some databases write out files with owner read permissions only, but c
```
Clean the generated tolls to match the tuples present in the position reports.
```
datafixtype3.py <output_dir>/my.data.out <output_dir>/my.tolls.out <output_dir>/my.tolls.clean
```

**Recap of scripts and order of usage:**

> On each raw file:
```
dataval.py <raw_file> <temp_file_1>
datarm2.py <temp_file_1> > <temp_file_2>
datamakeexit.py <temp_file_2> > <temp_file_3>
```
> Using the cleaned files create a single file:
```
datacombine.py <dir_of_cleaned_files>/ <output_dir>/clean.combined
```
> On the single combined file:
```
combine.py <output_dir>/clean.combined <output_dir> <num_xways>
```
> On the output toll file:
```
datafixtype3.py <output_dir>/my.data.out <output_dir>/my.tolls.out <output_dir>/my.tolls.clean
```
### Final outputs
The final outputs will be: 
```
<output_dir>/my.data.out
<output_dir>/my.tolls.clean
```
The scripts `preprawdata.sh` and `prepcleandata.sh` combine all the scripts and take a directory of raw or clean files, respectively, and output the final files.
