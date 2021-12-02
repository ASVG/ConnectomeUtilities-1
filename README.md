## Connectome utilities
Provides utility to look up and analyze connectivity between neurons in a circuit model.

  - circuit_models: Provides data loaders on top of bluepy to load data specific to 
  BlueBrain models in sonata format.
    - circuit_models.neuron_groups: Provies utility to load the properties of neurons in
    a circuit model, and group them according to various schemes:
      - by values of their properties (by etype, layer, mtype...)
      - by binned values of their properties (by coordinates)
      - by grouping them into a 2d grid (most useful in conjunction with flat coordinates)
    In addition to the default properties that already bluepy can load, it can add flat 
    mapped coordinates, supersampled or not, and neuron depth. It can further add properties
    looked up from any voxel atlas (looking it up by neuron location) and membership in any
    predefined group of neurons as a boolean property. For details, see below.

    - circuit_models.connection_matrix: Provides useful ways of looking up connection matrices.
    Either just a single matrix or matrices between groups of neurons. Or the counts of 
    connections between groups of neurons (reduced resolution connectome). 
    Can use any of the very flexible grouping schemes given by circuit_models.neuron_groups.

  - analysis: Provides functionality for simplifying and automating the analysis of connection matrices. 
  While the basic, atomic analyses are _not_ part of this repository, it provides functionality for
  easily applying it to multiple matrices, to submatrices defined by neuron properties (such as layer, mtypes, etc.)
  and comparing it to random controls.

  - flatmapping: Provides functionality on top of existing flat maps. That is, it does NOT
  provide the ability to create flat map volumes, but to get more out existing flat maps.
  This begins with creating "region images", i.e. images of flattened versions of a circuit,
  colored by region identity and extends to supersampling a flat map, i.e. going beyond its
  pixel resolution and even turning these arbitrary pixel coordinates into a um-based
  coordinate system.

  - io: Provides functionality to save / load multiple connection matrices into a single hdf5
  file. The matrices must be the values of a pandas.Series, indexed in any way.
  When such a Series is loaded again using this functionality, a lazy loading scheme is employed,
  i.e. the actual underlying matrices are only loaded when they are first actually accessed.
  This can be useful if you are only interested in a small number of the matrices in the Series.

### Installation
Simply run "pip install ."
Can then imported as "conntility".

### Using it
We can conceptually split a connectivity analysis into several parts:
  - **Loading neurons**: Start by looking up the neurons in a circuit model and their properties
  - **Filtering neurons**: Next, find the neurons you are really interested in by filtering out the uninteresting ones.
  - **Grouping neurons**: This step is optional. But maybe you want to analyze the result with respect to separate
  groups of neurons, such as per layer or mtype.
  - **Loading connectivity**: Next, load the connectivity between the neurons, yielding one or several connection
  matrices
  - **Analyzing connectivity**: Finally, run your analysis.

conntility provides python functions to simplify and standardize any of these steps. It also provides an object-oriented
layer on top of these functions that makes productionizing such analysis campaigns even easier. In that layer,
the details of each individual steps are detailed through .json-formatted configurations files. I will first describe
the use of this approach before describing the lower-level functionality.

#### Loader configuration
A loader configuration is an abstract description of a group of neurons, and a list of properties of interest. That is,
it allows you to describe which neurons you want to analyze, and which of their properties are relevant for your analysis.
The file is .json formatted containing up to three entries: *loading*, *filtering* and *grouping*. Any of these entries is
optional. 

Let's begin with a simple example of *loading* and *filtering* only:
```
{
    "loading": {
        "properties": ["x", "y", "z", "etype", "mtype", "layer"],
    },
    "filtering": [
        {
            "column": "etype",
            "values": ["bNAC", "cNAC"]
        },
        {
            "column": "layer",
            "value": 3
        }
    ]
}
```
Here, the *loading* block contains a singe entry "*properties*" that specifies the list of neuron properties to load -
the x, y, z coordinates, their e-types, m-types and layers. What other properties are available? Unfortunately, that 
depends on the circuit model - different models expose different properties. For the SSCx-v7 model this list is:
```
'etype', 'exc_mini_frequency', 'inh_mini_frequency', 'layer',
'me_combo', 'morph_class', 'morphology', 'mtype', 'orientation',
'region', 'synapse_class', 'x', 'y', 'z'
```
In addition to these basic properties directly exposed by the circuit model, four more *derived properties* are available:
```
'flat_x', 'flat_y', 'ss_flat_x', 'ss_flat_y'
```
*flat_x*, *flat_y* are the x, y coordinates in a flattened coordinate system (top-down view). They are given in an integer-valued
coordinate system with no specific units. *ss_flat_x*, *ss_flat_y* are given in a float-valued coordinate-sysyem in units 
that are roughly one micrometer. In order to load the derived properties certain prerequisites need to be fulfilled:
  - For *flat_x*, *flat_y* a **flatmap** atlas needs to exist in the circuits atlas directory. Additionally, the basic properties
  'x', 'y', 'z' must be loaded.
  - For *ss_flat_x*, *ss_flat_y* a **flatmap** atlas and an **orientation** atlas both need to exist in the circuits atlas directory.

**Note**: "properties" and in fact the entire "loading" block are optional. If you leave it out, then *all* available properties are
going to be loaded.


Next, the *filtering* block describes the subset of neurons you are interested. It is a list of of conditions and neurons failing
*any* of the conditions will be filtered out. That is, in the example above only "bNAC" and "cNAC" neurons in layer 3 will remain!

The subset is described by specifying the neuron property based on which you want to filter ("column") and the range
of valid values (here: "values"). The "*column*" given **must** be one of the properties that is specified in "loading". The valid
values can be specified in several ways:
  - A list of valid values:
  ```
  "values": *list*
  ```
  - A single valid value:
  ```
  "value": *scalar value*
  ```
  - For numerical properties: A valid interval:
  ```
  "interval": [*min value*, *max value*]
  ```

#### Using a loader config to load a connection matrix
Let's use our example loader config above to load a connection matrix:
```
import bluepy
from conntility.connectivity import ConnectivityMatrix

CIRC_FN =  "/gpfs/bbp.cscs.ch/project/proj83/circuits/Bio_M/20200805/CircuitConfig"
circ = bluepy.Circuit(CIRC_FN)

M = ConnectivityMatrix.from_bluepy(circ, loader_cfg)

[...]

print(M)
<conntility.connectivity.ConnectivityMatrix at 0x7ff4d1236700>

```

Here, "*loader_cfg*" is assumed to be either the path to a json file holding the example above, or the example itself,
i.e. a dict object with the *loading* and *filtering* entries. Both are supported.

Extracting the connectivity will take a minute, but you have a progressbar to keep you company.

The resulting object, *M* will give you access to the loaded neuron properties and the connectivity between the neurons.
First, we can list which properties have been loaded:
```
print(M.vertex_properties)
['etype' 'layer' 'mtype' 'x' 'y' 'z']
```
As specified by the loader config, we have the three coordinates, etype, mtype and layer. These can be directly accessed as
properties of the object:
```
print(M.etype)
['cNAC', 'cNAC', 'cNAC', 'cNAC', 'cNAC', ..., 'bNAC', 'bNAC', 'bNAC', 'bNAC', 'bNAC']
Length: 17570
Categories (11, object): ['bAC', 'bIR', 'bNAC', 'bSTUT', ..., 'cNAC', 'cSTUT', 'dNAC', 'dSTUT']

print(M.layer)
[3 3 3 ... 3 3 3]
```
As specified by the filtering, all neurons are in layer 3 and have an etype of either 'cNAC' or 'bNAC'.

The number of neurons is the length of the object:
```
print(len(M))
17570
```

##### Accessing matrices and submatrices
But what about the connectivity? It can be accessed as a sparse matrix, dense matrix or numpy.array as a property of the object.
The order of the rows / columns of the matrix matches the order of the entries of the loaded properties (layer, etype, etc.)
```
M.matrix 
<17570x17570 sparse matrix of type '<class 'numpy.bool_'>'
        with 17483 stored elements in COOrdinate format>

M.dense_matrix
matrix([[False, False, False, ..., False, False, False],
        [False, False, False, ..., False, False, False],
        [False, False, False, ..., False, False, False],
        ...,
        [False, False, False, ..., False, False, False],
        [False, False, False, ..., False, False, False],
        [False, False, False, ..., False, False, False]])

M.array
array([[False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False],
       ...,
       [False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False]])
```
You always have access to a global identifier of each of the neurons in the matrix:
```
print(M.gids)
[462330 462334 462341 ... 522514 522516 522517]

``` 
You can use the gids to define subpopulations or directly access submatrices:
```
print(M.subpopulation(M.gids[1000:2000]))
<conntility.connectivity.ConnectivityMatrix object at 0x7fffcb6fa9a0>  # object defining the subpopulation

print(len(M.subpopulation(M.gids[1000:2000])))
1000  # Because 2000 - 1000 = 1000

A = M.subpopulation(M.gids[1000:2000]).matrix  # matrix of the subpopulation
B = M.submatrix(M.gids[1000:2000])  # or direct access to the submatrix
print((A == B).mean())
1.0  # They are identical.
```

##### Subpopulations based on properties
Still, defining subpopulations by gid is a rather manual process. You first have to figure out which gids you are interested in
after all. There is functionality to help you with that. You can perform filtering based on neuron properties:
```
subpopulation = M.index("etype").eq("bNAC")  # Define a subpopulation based on "etype" and return the one equal to "bNAC"
print(len(subpopulation))  # Only 7212 of them are bNAC
7212

print(subpopulation.matrix)  # And the matrix of the subpopulation 
<7212x7212 sparse matrix of type '<class 'numpy.bool_'>'
        with 2474 stored elements in COOrdinate format>
```
This functionality works by first specifying which property you want to use in the call to .index; then give the range of valid values.
In a nutshell, it works similar to the *filtering* keyword in the loader config.
The range of valid values can be specified in different ways, each corresponding to a different function call:
  - .eq: *equal*. See above, matches a specified value. Works like using "value" in the *filtering* stage of a loader config
  - .isin: Matches a list of valid values. Works like using "values" in *filtering*.
  - .lt: *less than*. Only works with numerical values
  - .le: *less or equal*
  - .gt: *greater than*
  - .ge: *greater or equal*.
The equivalent of using "interval" in *filtering* is a chain of .lt and .ge.

##### Random subpopulations
Additionally, you can generate *constrained random subpopulations* that can serve as stochastic controls. For example:
```
indexer = M.index("mtype")
reference = M.subpopulation(M.gids[:1000])  # As an example, simply the first 1000 neurons.
rnd_sample = indexer.random_categorical(reference.gids)
print(len(rnd_sample))
1000
```
generates a random subsample of *M* with the same length as the reference_gids (here simply the first 1000 gids). Additionally, the random 
sample is constrained in the its distribution of "*mtype*" values will match the reference distribution!
```
(reference.mtype.value_counts() == rnd_sample.mtype.value_counts()).all()
True
```
For categorical properties use .random_categorical. For numerical values use .random_numerical. In that case, the distribution of the numerical value in discrete bins will match the reference. The number of bins can be specified:
```
rnd_sample = M.index("x").random_numerical(reference.gids, n_bins=10)
rnd_sample.x.mean(), rnd_sample.x.std()
(4389.180429069136, 1138.2503319186505)

reference.x.mean(), reference.x.std()
(4390.739357584497, 1143.8032496901749)  # Approximate match
```

#### More features of the loader configuration
Additionally, the following entries can be added to a loader config. They are of course all optional:
**base-target**
```
{
    "loading": {
        "base_target": "central_column_4_region_700um", 
    [...]
}
```
The *base_target* limits the neurons to a pre-defined group. This is, of course, similar to filtering. It acts differently though in that the target needs to have been pre-defined, and the filtering is applied *before* anything is loaded. Thus, it can be faster than applying a filter afterwards.

**atlas**
```
{
    "loading": {
        "atlas": [
            {"data": "distance", "properties": ["distance"]}
        ],
    [...]
```
The *atlas* loads additional data from one or several voxel atlases in .nrrd format. Value is a list of dicts that each specify the atlas to use ("data") and what to call the resulting property or properties ("properties"). What this does is to look up for each neuron the value in the atlas at the neurons x, y, z location. Therefore, to use this feature, the "x", "y" and "z" properties **must** also be loaded!

Note that the value of "properties" must be a list. This is because a voxel atlas can associate each voxel with more than one value (multi-dimensional atlas). The number of entries in the list must match the number of values associated with a voxel in the atlas.

**groups**
```
{
    "loading": {
        "groups":[
            {
                "filtering": [{"column": "etype", "values": ["bNAC", "cNAC"]}],
                "name": "is_bNAC_cNAC"
            }
        ]
    [...]
```
This associates an additional property with the neurons based on whether they pass afiltering check or not. The syntax for the entry under "filtering" is equal for the syntax of "filtering" at the root level. And it works in the same way: It defines a list of tests that a neuron passes or not. Only that non-passing neurons are not removed. Instead, a new property is added to the neurons, which is *True* if all checks are passed and False otherwise. This can be thought of as ad-hoc defined groups of neurons, where the value of the property denotes membership in the group. The name of the property to be added is explicitly set (*"name"*).

**include**
```
{
    "loading": {
        "groups":[
            {"include": "my_group.json"}
        ]
    [...]
```
The *include* keyword can be used at *any* location in a loader config. The way it works is very simple: If any element in the config is a dict with a single entry that is called "include", then the entire dict will be replaced with the contents of the .json file referenced by the value of "include". That is, if the contents of "my_group.json" is:
```
{
    "filtering": [{"column": "etype", "values": ["bNAC", "cNAC"]}],
    "name": "is_bNAC_cNAC"
}
```
Then the resulting behavior of the example using "include" is equal to the example above that.

The "include" allows you to re-use certain part of a config file that are useful without resorting to copy-paste. This can be used anywhere, but it is most useful for storing custom groups of neurons that cannot be readily assembled from their structural properties. Assemblies detected from simulated spiking activity is one possible use case.

To do this, simply perform "filtering" based on the column called "gid". The "gid" is a unique identifier of a neuron and is always loaded. Therefore, you can specify any group by giving a list of valid "gid"s:
```
# Contents of my_group.json
[
    {
        "filtering": [{"column": "gid", "values": 
            [521473, 468766, 477276, 483514, 495251, 499767, 520998, 474925,
                476827, 521806, 497750, 497824, 494201, 474007, 485759, 500136,
                502532, 470080, 482381, 477501]
                }],
        "name": "in_assembly1"
    },
    {
        "filtering": [{"column": "gid", "values": [517953, 485154, 479110, 482221, 485291, 506824, 469236, 488711,
            493455, 495871, 492482, 486045, 521636, 479584, 503382, 463251,
            521711, 488329, 463244, 502105]
            }],
        "name": "in_assembly2"
    }
]
```
Since the list of gids can grow very large it makes the file hard to read as a human. Therefore, it might be a good idea to keep these groups in a separate file that is referenced through an "include".


**Let's put all of that together**:
```
{
    "loading": {
        "base_target": "central_column_4_region_700um", 
        "properties": ["x", "y", "z", "etype", "mtype", "layer"],
        "atlas": [
            {"data": "distance", "properties": ["distance"]}
        ],
        "groups":[
            {
                "include": "my_group.json"
            },
            {
                "filtering": [{"column": "etype", "values": ["bNAC", "cNAC"]}],
                "name": "is_bNAC_cNAC"
            }
        ]
    },
    "filtering": [
        {
            "column": "etype",
            "values": ["bNAC", "cNAC"]
        },
        {
            "column": "layer",
            "value": 3
        }
    ]
}
```
```
M = ConnectivityMatrix.from_bluepy(circ, loader_cfg)
print(len(M))
1051
print(M.vertex_properties)
array(['etype', 'layer', 'mtype', 'x', 'y', 'z', 'distance',
       'in_assembly1', 'in_assembly2', 'is_bNAC_cNAC'], dtype=object)
```
We see that fewer neurons than in the earlier example are loaded, because we limited everything to the target "central_column_4_region_700um".

We also have additional vertex (neuron) properties: "distance" comes from the "atlas" property loaded, "in_assembly1", "in_assembly2" and "is_bNAC_cNAC" are boolean properties of group membership.

Now we can for example generate random control samples that match the "distance" distribution of our pre-defined assemblies:
```
assembly1 = M.index("in_assembly1").eq(True)
rnd_sample = M.index("distance").random_numerical(assembly1.gids)
```

#### Grouping in a loader config
So far, the general approach outlined was to load the entire connectivity matrix of a population, then access submatrices of interest using the .index function. There is one more tweak that allows you to define the submatrices you are interested in already in the loader config:
```
{
    "loading":
    [...]
    "filtering":
    [...]
    "grouping": [
        {
            "method": "group_by_properties",
            "columns": ["mtype", "etype"]
        }
    ]
}
```
The "grouping" keyword defines groups of neurons where you are interested in their submatrices. The value of "grouping" is a list, where each entry of the list yields a partition of the neurons into subgroups. The final groups used are then the product of the individual partitions, i.e. the intersections of all combinations of partitions.

The above example simply partitions neurons into groups based on their values of "mtype" and "etype", i.e. one group would be "L23_BP, bNAC", another "L23_NBC, cNAC". As a side note, this is equavalent, to a list of two separate groupings, one by "mtype", one by "etype" (but more compact):
```
{
    [...]
    "grouping": [
        {
            "method": "group_by_properties",
            "columns": ["mtype"]
        },
        {
            "method": "group_by_properties",
            "columns": ["etype"]
        }
    ]
}
```

#### Using a grouped loader config
```
from conntility.connectivity import ConnectivityGroup

G = ConnectivityGroup.from_bluepy(circ, loader_cfg)

print(G.index)
MultiIndex([( 'L23_BP', 'bNAC'),
            ( 'L23_BP', 'cNAC'),
            ('L23_BTC', 'bNAC'),
            ('L23_BTC', 'cNAC'),
            ('L23_CHC', 'cNAC'),
            ('L23_DBC', 'bNAC'),
            ('L23_LBC', 'bNAC'),
            ('L23_LBC', 'cNAC'),
            ( 'L23_MC', 'bNAC'),
            ( 'L23_MC', 'cNAC'),
            ('L23_NBC', 'bNAC'),
            ('L23_NBC', 'cNAC'),
            ('L23_NGC', 'bNAC'),
            ('L23_NGC', 'cNAC'),
            ('L23_SBC', 'bNAC')],
           names=['idx-mtype', 'idx-etype'])

print(G["L23_BP", "bNAC"])
<conntility.connectivity.ConnectivityMatrix at 0x7ff4cba65850>

print(G["L23_BP", "bNAC"].matrix.shape)
(70, 70)
```
As we can see, this results in an object that contains the subpopulations of interest. It can be indexed by the "mtype" and "etype" of the subpopulations, corresponding to the properties we have specified in the "grouping" of the loader config. Indexing returns a representation of the subpopulation with all the features described above.

At this point, this is all the *"ConnectivityGroup"* can do, but more features are planned in the future.
