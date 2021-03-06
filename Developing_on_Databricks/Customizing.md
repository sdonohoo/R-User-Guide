
# Customizing with Init Scripts

In addition to the various flavors of [Databricks Runtime](https://github.com/marygracemoesta/R-User-Guide/blob/master/Databricks_Architecture_Overview/DB_Runtime.md#databricks-runtimes) that can be selected at cluster creation, users can supply initialization scripts to bootstrap the environment in various ways.  This document will serve as a repository to easily access scripts that perform various tasks. 


##### Contents

* [Creating & Attaching Init Scripts](#creating-&-attaching-init-scripts)
* [Example Scripts](#example-scripts)
  * [RStudio Server Installation](#rstudio-server-installation)
  * [Apache Arrow Installation](#apache-arrow-installation)
  * [Library Installation](#library-installation)
  * [Running an R Script](#running-an-r-script)
  * [Modifying Rprofile in RStudio](#modifying-rprofile)

____

### Creating & Attaching Init Scripts

An initialization script is simply a bash script on DBFS that is executed at cluster startup.  These are fairly flexible, and you can learn more in the official documentation [here](https://docs.databricks.com/clusters/init-scripts.html#init-script-types).  You can upload the file to DBFS through the Databricks CLI, or you can write the file to DBFS using `dbutils` in a Databricks Notebook.  The examples below will be using `dbutils` in Python. 

Once you have the script and are ready to attach it to a cluster, you'll need to edit the Advanced Options section of the cluster UI.  Under the _Init Scripts_ tab, you'll be able to specify a path to the script on DBFS.  

<img src="https://github.com/marygracemoesta/R-User-Guide/blob/master/Getting_Started/images/init_script_example.png?raw=true" height=600 width=500>

Now the cluster will run the script every time it is restarted.

### Example Scripts

#### RStudio Server Installation

To work with RStudio on Databricks, you'll need to run the following code in a **Python** cell in a notebook. 

```python
%python

## Define the contents of the script
script = """
  if [[ $DB_IS_DRIVER = "TRUE" ]]; then
    sudo apt-get update
    sudo apt-get install -y gdebi-core alien
    cd /tmp
    sudo wget https://download2.rstudio.org/rstudio-server-1.1.453-amd64.deb
    sudo gdebi -n rstudio-server-1.1.453-amd64.deb
    sudo rstudio-server restart
    exit 0
  else
    exit 0
  fi
"""

## Create directory to save the script in
dbutils.fs.mkdirs("/databricks/rstudio")

## Save the script to DBFS
dbutils.fs.put("/databricks/rstudio/rstudio-install.sh", script, True)
```

Note the path is `/databricks/rstudio/rstudio-install.sh`.  This is what we will add to the _Init Script_ path in the Advanced Options section of the cluster UI.


#### Apache Arrow Installation

Apache Arrow is an open source project that provides a common in-memory format between different processes.  This is especially useful when moving data back and forth between Spark and R, as you would with [user defined functions](linktocome).  To enable Arrow with R on Databricks, the first step is to attach an init script to a cluster.  Using a Python cell in a Databricks Notebook, run the following cell:

```python
%python

## Define contents of the script
script = """
#!/bin/bash
sudo apt update
sudo apt install -y -V apt-transport-https lsb-release
curl https://dist.apache.org/repos/dist/dev/arrow/KEYS | sudo apt-key add -
sudo tee /etc/apt/sources.list.d/apache-arrow.list <<APT_LINE
deb [arch=amd64] https://dl.bintray.com/apache/arrow/$(lsb_release --id --short | tr 'A-Z' 'a-z')/ $(lsb_release --codename --short) main
deb-src https://dl.bintray.com/apache/arrow/$(lsb_release --id --short | tr 'A-Z' 'a-z')/ $(lsb_release --codename --short) main
APT_LINE
sudo apt update
sudo apt install -y -V libarrow-dev # For C++
sudo apt install -y -V libarrow-glib-dev # For GLib (C)
sudo apt install -y -V libgandiva-dev # For Gandiva C++
sudo apt install -y -V libgandiva-glib-dev # For Gandiva GLib (C)
sudo apt install -y -V libparquet-dev # For Apache Parquet C++
sudo apt install -y -V libparquet-glib-dev # For Apache Parquet GLib (C
"""

## Create directory to save the script in
dbutils.fs.mkdirs("/databricks/arrow")

## Save the script to DBFS
dbutils.fs.put("/databricks/arrow/arrow-install.sh", script, True)
```

Note the path is `/databricks/arrow/arrow-install.sh`. This is what we will add to the Init Script path in the Advanced Options section of the cluster UI.  For more details please see the [Apache Arrow with R](https://github.com/marygracemoesta/R-User-Guide/blob/master/Spark_Distributed_R/arrow.md) section.

#### Library Installation
One quick way to install a list of packages on a cluster is through an init script.  At a certain point you may see long cluster startup times as R has to download, compile, and install the packages.  See the [Faster Package Loads](https://github.com/marygracemoesta/R-User-Guide/blob/master/Developing_on_Databricks/package_management.md#faster-package-loads) section for an alternative solution.

Using a Python cell in a Databricks Notebook run the following code, making sure to add your list of packages using `install.packages()` as shown.

```python
%python

## Define contents of script 
script = """
#!/bin/bash
R --vanilla <<EOF 
install.packages('forecast', repos="https://cran.microsoft.com/snapshot/2017-09-28/")
install.packages('sparklyr', dependencies=TRUE, repos="https://cran.microsoft.com/snapshot/2017-09-28/")
q()
EOF
"""

## Create directory to save the script in
dbutils.fs.mkdirs("/databricks/rlibs")

## Save the script to DBFS
dbutils.fs.put("/databricks/rlibs/r-library-install.sh", script, True)
```

Note the path is `/databricks/rlibs/r-library-install.sh`.  This is what we will add to the _Init Script_ path in the Advanced Options section of the cluster UI.

#### Running an R Script
If you want to run an arbitrary R script on cluster startup, you can do that too. Using a Python cell in a Databricks Notebook, run the following code, replacing `<INSERT R SCRIPT>` with ... your R script!

```python
%python

## Define contents of script 
script = """
#!/bin/bash
R --vanilla <<EOF 
<INSERT R SCRIPT>
q()
EOF
"""

## Create directory to save the script in
dbutils.fs.mkdirs("/databricks/rscripts")

## Save the script to DBFS
dbutils.fs.put("/databricks/rscripts/my-r-script.sh", script, True)
```

Note the path is `/databricks/rscripts/my-r-script.sh`.  This is what we will add to the _Init Script_ path in the Advanced Options section of the cluster UI.

#### Modifying Rprofile in RStudio
Customizing your Rprofile is great way to enhance your R experience.  You can recreate this experience on Databricks by leveraging an init script to modify your `Rprofile.site` file.  Essentially you will pipe an R script into the `Rprofile.site` file, appending changes to it.  **Note:**  This will only work on RStudio, not in an R Notebook.  

As a prerequisite, upload a R file containing the modifications you'd like to make to DBFS.  In this example the file is called `r_profile_changes.R`  Then run the following code in Databricks Notebook using Python:

```python
%python

## Define contents of script
script = """
#!/bin/bash
if [[ $DB_IS_DRIVER = "TRUE" ]]; then
echo 'source("/dbfs/R/scripts/r_profile_changes.R")' >> /usr/lib/R/etc/Rprofile.site
fi
"""

## Create directory to save the script in
dbutils.fs.mkdirs("/databricks/rscripts")

## Save the script to DBFS
dbutils.fs.put("/databricks/rscripts/modify-rprofile.sh", script, True)
```
Note the path is `/databricks/rscripts/modify-rprofile.sh`.  This is what we will add to the _Init Script_ path in the Advanced Options section of the cluster UI.

___
[Back to table of contents](https://github.com/marygracemoesta/R-User-Guide#contents)
