### dcarrillox edits
Currently, the path to the sqlite database file is hardcoded in `CrisprOpenDB_HostID.py` (lines 102-103) so it needs
to be in `CrisprOpenDB/SpacersDB/` as stated in the README.md . :

~~~
if self._connection is None:
    self._connection = CrisprOpenDB.CrisprOpenDB(os.path.join("CrisprOpenDB", "SpacersDB", "CrisprOpenDB.sqlite"))
~~~

This is a problem since I want to put CrisprOpenDB in a Singularity container and `CrisprOpenDB.sqlite` is too heavy for that.
I modify the code so the sqlite file can be outside of the repo. `blastdb` and `fastadb` files should not be a problem since there already
options for specifying their paths.

**Code modifications:**
- `CL_Interface.py`
  - Line 16: added option `-q` so user can specify path to `CrisprOpenDB.sqlite`
  - Line 38, old 37: added `args.sqlitedb` as argument for `PhageHostFinder()`

- `CrisprOpenDB_HostID.py`
  - Lines 12-17: added `sqlite_db` to the definition of the Class
  - Line 104, old 103: change the hardcoded path to the variable `sqlite_db`

It worked with this test:

~~~
$ python CrisprOpenDB/CL_Interface.py -i  CrisprOpenDB/TestGenomes/KJ489400.fasta -b SpacersDB -q CrisprOpenDB.sqlite
('KJ489400.1', 'Bacillus', 1)
~~~

#### Singularity
I create a Singularity container with the modifications. Definition file `CrisprOpenDB.def` for building the container is also in the repository.
To do so:

~~~
$ sudo singularity build crisprpendb.sif CrisprOpenDB.def
~~~

Container is also available from `library://dcarrillo/default/crispropendb:0.1` .

**IMPORTANT**: blastdb has to be built with the `makeblastdb` provided in the container, which is `blast 2.10.1`. This is to avoid version
incompatibility issues. To do so:

~~~
$ singularity run library://dcarrillo/default/crispropendb:0.1 makeblastdb -in SpacersDB.fasta -out SpacersDB -dbtype nucl
~~~

---

# CrisprOpenDB

This command-line host prediction tool was implemented to run predictions for large numbers of phage genomes and to offer a more customized host prediction process. The method is described in the following paper, that you should cite in publications:
> Moïra B Dion, Pier-Luc Plante, Edwige Zufferey, Shiraz A Shah, Jacques Corbeil, Sylvain Moineau, Streamlining CRISPR spacer-based bacterial host predictions to decipher the viral dark matter, *Nucleic Acids Research*, Volume 49, Issue 6, 6 April 2021, Pages 3127–3138, https://doi.org/10.1093/nar/gkab133

### Prerequisites

First, download or clone this repository. The easiest way is using `git clone https://github.com/plpla/CrisprOpenDB.git`.

Next, we recommend to create a virtual environment to run the tool. You can set up a conda environment using the `conda_env.txt` file in the initial directory and the following command:
```python
conda create -c conda-forge --name CrisprOpenDB_env --file conda_env.txt
```

Don't forget to activate the environment to use it: `conda activate CrisprOpenDB_env`.
You can now install the tool: `python setup.py install`.

To use this program, you need to download the spacer database and the sqlite file (http://crispr.genome.ulaval.ca/dash/PhageHostIdentifier_DBfiles.zip) and unzip the files in the `CrisprOpenDB/SpacersDB/` directory. Note that files are quite large. The download size is about 800Mo for the compressed file. Once unzipped, file sizes will be approximately 600Mo for the spacer database and 3.8Go for the sqlite file. If you plan on using Blast to run the program, you need to build the blast database. Instructions for this optional step are at the end of this README file with instructions on how to install Blast if it not already installed on your computer.

Once these steps are complete, don't forget to go back to the initial directory to run the program. You can now try your installation with one of our test genome, as explained below.

### Running

To run the program, you must launch the `CL_Interface.py` file in the working directory.
Here is an example of how to run the program using a *Salmonella* phage genome and a number of mismatches of 2:
```python
python CL_Interface.py -i Salmonella_161.fasta -m 2
```
### Test genomes

We provide test genomes for each level of prediction in the `TestGenomes/` directory. These will allow you to make sure the tool has been properly set up before moving on with your personal analyses.
The commands you should use to perform the tests as well as the expected results are shown down below.

Level 1 prediction:
```python
python CL_Interface.py -i TestGenomes/KJ489400.fasta
```
`('KJ489400.1', 'Bacillus', 1)`

Level 2 prediction:
```python
python CL_Interface.py -i TestGenomes/AY133112.fasta
```
`('AY133112.1', 'Vibrio', 2)`

Level 3 prediction:
```python
python CL_Interface.py -i TestGenomes/MT074469.fasta
```
`('MT074469.1', 'Salmonella', 3)`

Level 4 prediction (note that level 4 predictions do not return a genus):
```python
python CL_Interface.py -i TestGenomes/MT074470.fasta
```
`('MT074470.1', 'Enterobacterales', 4)`

### Options and additional information for installation

Alignment can be done using `blastn` or `fasta36`. To install Blast inside the Anaconda environment run:
```
conda activate CrisprOpenDB_env
conda install -c bioconda blast
```

If using BLAST, you also need to use `makeblastdb` before running the phage host identification tool. Here is the command line you should use when running `makeblastdb` from the `CrisprOpenDB/SpacersDB/` directory:
```
makeblastdb -in SpacersDB.fasta -dbtype nucl -out SpacersDB
```
If you wish, you can also provide your own BLAST or FASTA database to perform the alignment.

If you would like to manually further explore a prediction, you can use the `-t, --table` option. This will send the table containing both blast results and spacers information extracted from the spacers database to an external CSV file. Since one file per phage genome is issued, we recommend that you do not use this option when first running the tool (especially with large datasets), but rather use it afterwards if you need further information on a specific genome.
