# ServiceX Client Library

 Client access library for ServiceX

[![GitHub Actions Status](https://github.com/ssl-hep/ServiceX_frontend/workflows/CI/CD/badge.svg?branch=master)](https://github.com/ssl-hep/ServiceX_frontend/actions)
[![Code Coverage](https://codecov.io/gh/ssl-hep/ServiceX_frontend/graph/badge.svg)](https://codecov.io/gh/ssl-hep/ServiceX_frontend)

[![PyPI version](https://badge.fury.io/py/servicex.svg)](https://badge.fury.io/py/servicex)
[![Supported Python versions](https://img.shields.io/pypi/pyversions/servicex.svg)](https://pypi.org/project/servicex/)

## Introduction

Given you have a selection string, this library will manage submitting it to a ServiceX instance and retrieving the data locally for you.
The selection string is often generated by another front-end library, for example:

- [func_adl.xAOD](https://github.com/iris-hep/func_adl_xAOD) (for ATLAS xAOD's)
- [func_adl.uproot](https://github.com/iris-hep/func_adl.uproot) (for flat ntuples)
- [tcut_to_castle](https://pypi.org/project/tcut-to-qastle/) (translates `TCut` like syntax into a `servicex` query - should work for both)

## Prerequisites

Before you can use this library you'll need:

- An environment based on python 3.7 or later. 3.11 is highest supported at the moment.
- A `ServiceX` end-point. This is usually gotten by logging into and getting approved at the servicex endpoint. Once you do that, you'll have an API token, which this library needs to access the `ServiceX` endpoint, and the web address where you got that token (the `endpoint` address).

### How to access your endpoint

The API access information is normally placed in a configuration file (see the section below). Create a config file, `servicex.yaml`, in the `yaml` format, in the appropriate place for your work that contains the following (for the `xaod` backend; use `uproot` for the `type` for the uproot backend):

```yaml
api_endpoints:
  - name: <your-endpoint-name>
    endpoint: <your-endpoint>
    token: <api-token>
    type: xaod
    
```

All strings are expanded using python's [os.path.expand](https://docs.python.org/3/library/os.path.html#os.path.expandvars) method - so `$NAME` and `${NAME}` will work to expand existing environment variables.

You can list multiple end points by repeating the block of dictionary items, but using a different name.

Finally, you can create the objects `ServiceXAdaptor` and `MinioAdaptor` by hand in your code, passing them as arguments to `ServiceXDataset` and inject custom endpoints and credentials, avoiding the configuration system. This is probably only useful for advanced users.

These config files are used to keep confidential credential information - so that it isn't accidentally placed in a public repository.

If no endpoint is specified or config file containing a useful endpoint is found, then the library defaults to the developer endpoint, which is `http://localhost:5000` for the web-service API. No passwords are used in this case.

## Usage

The following lines will return a `pandas.DataFrame` containing all the jet pT's from an ATLAS xAOD file containing Z->ee Monte Carlo:

```python
    from servicex import ServiceXDataset
    query = "(call ResultTTree (call Select (call SelectMany (call EventDataset (list 'localds:bogus')) (lambda (list e) (call (attr e 'Jets') 'AntiKt4EMTopoJets'))) (lambda (list j) (/ (call (attr j 'pt')) 1000.0))) (list 'JetPt') 'analysis' 'junk.root')"
    dataset = "mc15_13TeV:mc15_13TeV.361106.PowhegPythia8EvtGen_AZNLOCTEQ6L1_Zee.merge.DAOD_STDM3.e3601_s2576_s2132_r6630_r6264_p2363_tid05630052_00"
    ds = ServiceXDataset(dataset, backend_name=`xaod`)
    r = ds.get_data_pandas_df(query)
    print(r)
```

And the output in a terminal window from running the above script (takes about 1-2 minutes to complete):

```bash
python scripts/run_test.py http://localhost:5000/servicex
            JetPt
entry
0       38.065707
1       31.967096
2        7.881337
3        6.669581
4        5.624053
...           ...
710183  42.926141
710184  30.815709
710185   6.348002
710186   5.472711
710187   5.212714

[11355980 rows x 1 columns]
```

If your query is badly formed or there is an other problem with the backend, an exception will be thrown with information about the error.

If you'd like to be able to submit multiple queries and have them run on the `ServiceX` back end in parallel, it is best to use the `asyncio` interface, which has the identical signature, but is called `get_data_pandas_df_async`.

For documentation of `get_data` and `get_data_async` see the `servicex.py` source file.

The `backend_name` tells the library where to look in the `servicex.yaml` configuration file to find an end point (url and authentication information). See above for more information.

### How to specify the input data

How you specify the input data, and what data can be ingested, is ultimately defined by the configuration of the `ServiceX` backend you are running against. This `servicex`
library supports the following:

- A Dataset Identifier (DID): For example, `rucio://mc16a_13TeV:my_dataset`, or `cernopendata://1507`, both of which are resolved to a list of
files (in one case, a set of ATLAS data files, and in the other some CMS Run 1 AOD files).
- A single file located at a `http` or `root` endpoint: For example, `root://myfile.root` or `http://myfile.root`. ServiceX must be able to
access these files without special permissions.
- A list of files located at `http` or `root` endpoints: For example, `[root://myfile1.root, http://myfile2.root]`. ServiceX must be able to
access these files without special permissions.
- [depreciated] A bare (DID): this is an unadorned identifier, and is routed to the backend's default DID resolver. The default
is defined at runtime. It is depreciated because a backend configuration change can break your code.

### The Local Data Cache

To speed things up - especially when you run the same query multiple times, the `servicex` package will cache queries data that comes back from Servicex. You can control where this is stored with the `cache_path` in the configuration file (see below). By default it is written in the temp directory of your system, under a `servicex_{USER}` directory. The cache is unbound: it will continuously fill up. You can delete it at any time that you aren't processing data: data will be re-downloaded or re-transformed in `ServiceX`.

There are times when you want the system to ignore the cache when it is running. You can do this by using `ignore_cache()`:

```python
from servicex import ignore_cache

with ignore_cache():
  do_query():
```

If you are using a Jupyter notebook, the `with` statement can't really span cells. So use `ignore_cache().__enter__()` instead. Or you can do something like:

```python
from servicex import ignore_cache

ic = ignore_cache()
ic.__enter__()

...

ic.__exit__(None, None, None)
```

If you wish to disable the cache for a single dataset, use the `ignore_cache` parameter when you create it:

```python
ds = ServiceXDataset(dataset, ignore_cache=True)
```

Finally, you can ignore the cache for a dataset for a short period of time by using the same context manager pattern:

```python
ds = ServiceXData(dataset)
with ds.ignore_cache():
  do_query(ds)  # Cache is ignored
do_query(ds)  # Cache is not ignored
```

#### Analysis And Query Cache

The `servicex` library can write out a local file which will map queries to backend `request-id`'s. This file can then be used on other people, checked into repositories, etc., to reference the same data in the backend. The advantage is that the backend does not need to re-run the query - the `servicex` library need only download it again. When a user uses multiple machines or shares analysis code with an analysis team, this is a much more efficient use of resources.

- By default the library looks for a file `servicex_query_cache.json` in the current working directory, or a parent directory of the current working directory.
- To trigger the creation and updating of a cache file call the function `update_local_query_cache()`. If you like you can pass in a filename/path. By default it will use `servicex_query_cache.json` in the local directory. The file will be both used for look-ups and will be updated with all subsequent queries. Except under very special cases, it is suggested that one users the filename `servicex_query_cache.json`.
- You can also create the file by using the bash command `touch servicex_query_cache.json` - if you are using the default name.
- If that file is present when a query is run, it will attempt to download the data from the endpoint, only resubmitting the query if the endpoint doesn't know about the query. As long as the file `servicex_query_cache.json` is in the current working directory (or above), it will be picked up automatically: no need to call `update_local_query_cache()`.

The cache search order is as follows:

- The analysis query cache is searched first.
- If nothing is found there, then the local query cache is used next.
- If nothing is found there, then the query is resubmitted.

_Note_: Eventually the backends will contain automatic cache lookup and this feature will be much less useful as it will occur automatically, on the backend.

#### Deleting Files from the local Data Cache

It is not recommended to alter the cache. The software expects the cache to be in a certain state, and randomly altering it can lead to unexpected behavior.

Besides telling the `servicex` library to ignore the cache in the above ways, you can also delete files from the local cache.
The local cache directory is split up into sub-directories. Deleting files from each of the directories:

- `query_cache` - this directory contains the mapping between the query text (or its hash) and the ServiceX backend's `request-id`. If you delete a file from here, it is as if the query was never made, and is the same as using the ignore methods above.
- `query_cache_status` - contains the last retrieved status from the backend. Deleting this will cause the library to refresh the missing status. This file is updated continuously until the query is completed.
- `file_list_cache` - Each file contains a json list of all the files in the `minio` bucket for a particular request id. Deleting a file from this directory will cause the frontend to re-download the complete list of files (the file in this directory isn't created until all files have been downloaded).
-`data` - This directory contains the files that have been downloaded locally. If you delete a data file from this directory, it will trigger a re-download. Note that if the servicex endpoint doesn't know about the original query, or the minio bucket is missing, it will force the transform being re-run from scratch.

## Configuration

The `servicex` library searches for configuration information in several locations to determine what end-point it should connect to:

1. The config file can be called `servicex.yaml`, `servicex.yml`, or `.servicex`. The files are searched in that order, and all present are used.
1. A config file in the current working directory.
1. A config file in any working directory above your current working directory.
1. A config file in the user's home directory (`$HOME` on Linux and Mac, and your profile
   directory on Windows).
1. The `config_defaults.yaml` file distributed with the `servicex` package.

The file can contain an `api_endpoint` as mentioned earlier. In addition the other following things can be put in:

- `cache_path`: Location where queries, data, and a record of queries are written. This should be an absolute path the person running the library has r/w access to. On windows, make sure to escape `\` - and best to follow standard `yaml` conventions and put the path in quotes - especially if it contains a space. Top level yaml item (don't indent it accidentally!). Defaults to `/tmp/servicex_<username>` (with the temp directory as appropriate for your platform) Examples:
  - Windows: `cache_path: "C:\\Users\\gordo\\Desktop\\cacheme"`
  - Linux: `cache_path: "/home/servicex-cache"`

- `backend_types` - a list of yaml dictionaries that contains some defaults for the backends. By default only the `return_data` is there, which for `xaod` is `root` and `uproot` is `parquet`. There is also a `cms_run1_aod` which returns `root`. Allows `servicex` to convert to `pandas.DataFrame` or `awkward` if requested by the user.

All strings are expanded using python's [os.path.expand](https://docs.python.org/3/library/os.path.html#os.path.expandvars) method - so `$NAME` and `${NAME}` will work to expand existing environment variables.

For non-standard use cases, the user can specify:
- The code generator that is used by the backend. This is done by passing a `codegen` argument to ServiceXDataset. This argument is normally inherited from the backend type set in `servicex.yaml`, but can be overridden with any valid `codegen` contained in the default type listing. A `codegen` entry can also be added to a backend in the yaml file to use as default.
- The type of backend, using the `backend_type` argument on ServiceXDataset. This overrides the backend type setting in the `servicex.yaml` file.

## Features

Implemented:

- Accepts a `qastle` formatted query
- Exceptions are used to report back errors of all sorts from the service to the user's code.
- Data is return in the following forms:
  - `pandas.DataFrame` an in process DataFrame of all the data requested
  - `awkward` an in process `JaggedArray` or dictionary of `JaggedArray`s
  - A list of root files that can be opened with `uproot` and used as desired.
  - Not all output formats are compatible with all transformations.
- Complete returned data must fit in the process' memory
- Run in an async or a non-async environment and non-async methods will accommodate automatically (including `jupyter` notebooks).
- Support up to 100 simultaneous queries from a laptop-like front end without overwhelming the local machine (hopefully ServiceX will be overwhelmed!)
- Start downloading files as soon as they are ready (before ServiceX is done with the complete transform).
- It has been tested to run against 100 datasets with multiple simultaneous queries.
- It supports local caching of query data
- It will provide feedback on progress.
- Configuration files supported so that user identification information does not have to be checked
  into repositories.

## Testing

This code has been tested in several environments:

- Windows, Linux, MacOS
- Python 3.7, 3.8, 3.9, 3.10
- Jupyter Notebooks (not automated), regular python command-line invoked source files

### Non-standard backends

When doing backend development, often ports 9000 and 5000 are forwarded to the local machine exposing the `minio` and `ServiceX_App` instances. In that case, you'll need to create a configuration file that has `http://localhost:5000` as the end point. No API token is necessary if the development `ServiceX` instance doesn't have authorization turned on.

## API

Everything is based around the `ServiceXDataset` object. Below is the documentation for the most common parameters.

```python
  ServiceXDataset(dataset: str,
                 backend_name: Optional[str] = None,
                 image: str = 'sslhep/servicex_func_adl_xaod_transformer:v0.4',
                 max_workers: int = 20,
                 result_destination = 'object-store',
                 servicex_adaptor: ServiceXAdaptor = None,
                 minio_adaptor: MinioAdaptor = None,
                 cache_adaptor: Optional[Cache] = None,
                 status_callback_factory: Optional[StatusUpdateFactory] = _run_default_wrapper,
                 local_log: log_adaptor = None,
                 session_generator: Callable[[], Awaitable[aiohttp.ClientSession]] = None,
                 config_adaptor: ConfigView = None):
  '''
      Create and configure a ServiceX object for a dataset.

      Arguments

          dataset                     Name of a dataset from which queries will be selected.
          backend_name                The type of backend. Used only if we need to find an
                                      end-point. If we do not have a `servicex_adaptor` then this
                                      will default to xaod, unless you have any endpoint listed
                                      in your servicex file. It will default to best match there,
                                      in that case.
          image                       Name of transformer image to use to transform the data
          max_workers                 Maximum number of transformers to run simultaneously on
                                      ServiceX.
          result_destination          Where the transformers should write the results.
                                      Defaults to object-store, but could be used to save
                                      results to a posix volume                                      
          servicex_adaptor            Object to control communication with the servicex instance
                                      at a particular ip address with certain login credentials.
                                      Default comes from the `config_adaptor`.
          minio_adaptor               Object to control communication with the minio servicex
                                      instance.
          cache_adaptor               Runs the caching for data and queries that are sent up and
                                      down.
          status_callback_factory     Factory to create a status notification callback for each
                                      query. One is created per query.
          local_log                   Log adaptor for logging.
          session_generator           If you want to control the `ClientSession` object that
                                      is used for callbacks. Otherwise a single one for all
                                      `servicex` queries is used.
          config_adaptor              Control how configuration options are read from the
                                      configuration file (servicex.yaml, servicex.yml, .servicex).

      Notes:

          -  The `status_callback` argument, by default, uses the `tqdm` library to render
             progress bars in a terminal window or a graphic in a Jupyter notebook (with proper
             jupyter extensions installed). If `status_callback` is specified as None, no
             updates will be rendered. A custom callback function can also be specified which
             takes `(total_files, transformed, downloaded, skipped)` as an argument. The
             `total_files` parameter may be `None` until the system knows how many files need to
             be processed (and some files can even be completed before that is known).
  '''
 ```

To get the data use one of the `get_data` method. They all have the same API, differing only by what they return.

```python
 |  get_data_awkward_async(self, selection_query: str, title: Optional[str] = None) -> Dict[bytes, Union[awkward.array.jagged.JaggedArray, numpy.ndarray]]
 |      Fetch query data from ServiceX matching `selection_query` and return it as
 |      dictionary of awkward arrays, an entry for each column. The data is uniquely
 |      ordered (the same query will always return the same order). If specified, the optional title is passed to the backend and can be viewed on the status page.
 |
 |  get_data_awkward(self, selection_query: str, title: Optional[str] = None) -> Dict[bytes, Union[awkward.array.jagged.JaggedArray, numpy.ndarray]]
 |      Fetch query data from ServiceX matching `selection_query` and return it as
 |      dictionary of awkward arrays, an entry for each column. The data is uniquely
 |      ordered (the same query will always return the same order).  If specified, the optional title is passed to the backend and can be viewed on the status page.
```

Each data type comes in a pair - an `async` version and a synchronous version.

- `get_data_awkward_async, get_data_awkward` - Returns a dictionary of the requested data as `numpy` or `JaggedArray` objects.
- `get_data_rootfiles`, `get_data_rootfiles_async` - Returns a list of locally download files (as `pathlib.Path` objects) containing the requested data. Suitable for opening with [`ROOT::TFile`](https://root.cern.ch/doc/master/classTFile.html) or [`uproot`](https://github.com/scikit-hep/uproot).
- `get_data_pandas_df`, `get_data_pandas_df_async` - Returns the data as a `pandas` `DataFrame`. This will fail if the data you've requested has any structure (e.g. is hierarchical, like a single entry for each event, and each event may have some number of jets).
- `get_data_parquet`, `get_data_parquet_async` - Returns a list of files locally downloaded that can be read by any parquet tools.

### Streaming Results

The `ServiceX` backend generates results file-by-file. The above API will return the list of files when the transform has completed. For large transforms this can take some time: no need to wait until it is completely done before processing the files!

- `get_data_rootfiles_stream`, `get_data_parquet_stream`, `get_data_pandas_stream`, and `get_data_awkward_stream` return a stream of local file path's as each result from the backend is downloaded. All take just the `qastle` query text as a parameter and return a python `AsyncIterator` of `StreamInfoData`. Note that files downloaded locally are cached - so when you re-run the same query it will immediately render all the `StreamInfoData` objects from the async stream with no waiting.
- `get_data_rootfiles_url_stream` and `get_data_parquet_url_stream` return a stream of URL's that allow direct access in the backend to the data generated as it is finished. All take just the `qastle` query text as a parameter, and return a python `AsyncIterator` of `StreamInfoUrl`. These methods are probably most useful if you are working in the same data center that the `ServiceX` service is running in.

The `StreamInfoURL` contains a `bucket`, `file`, and a `url` property. The `url` property can be used to access the requested data without authentication for about 24 hours (depends on the `ServiceX` backend's configuration). Use the `file` to understand what part of the starting dataset that data came from. And as this de-facto points to a `minio` database currently, the `bucket` can be used to find the host bucket name.

The `StreamInfoData` contains a `file` and a `path` property. The `file` is as above, and the `path` is a `pathlib.Path` object that points to the file that has been downloaded into the cache locally.

An example using the async interface that performs the same operation as the initial example above:

```python
    from servicex import ServiceXDataset
    query = "(call ResultTTree (call Select (call SelectMany (call EventDataset (list 'localds:bogus')) (lambda (list e) (call (attr e 'Jets') 'AntiKt4EMTopoJets'))) (lambda (list j) (/ (call (attr j 'pt')) 1000.0))) (list 'JetPt') 'analysis' 'junk.root')"
    dataset = "mc15_13TeV:mc15_13TeV.361106.PowhegPythia8EvtGen_AZNLOCTEQ6L1_Zee.merge.DAOD_STDM3.e3601_s2576_s2132_r6630_r6264_p2363_tid05630052_00"
    ds = ServiceXDataset(dataset)

    async for f in ds.get_data_rootfiles_stream(query):
      print(f.path)
```

Notes:

- `ServiceX` might fail part way through the transformation - so be ready for an exception to bubble out of your `AsyncIterator`!
- If you are combining different queries whose filtering is identical, make sure to use the `file` property to match results - otherwise you won't have an event-to-event matching!

## Development

For any changes please feel free to submit pull requests! We are using the `gitlab` workflow: the `master` branch represents the latests updates that pass all tests working towards the next version of the software. Any PR's should be based off the most recent version of `master` if they are for new features. Each release is frozen on a dedicated release branch, e.g. v2.0.0. If a bug fix needs to be applied to an existing release, submit a PR to master mentioning the affected version(s). After the PR is merged to master, it will be applied to the relevant release branch(es) using git cherry-pick.

To do development please setup your environment with the following steps:

1. A python 3.7 development environment
2. Fork/Pull down this package, XX
3. `python -m pip install -e .[test]`
4. Run the tests to make sure everything is good: `pytest`.

Then add tests as you develop. When you are done, submit a pull request with any required changes to the documentation and the online tests will run.

### To create a release branch

```bash
get checkout 2.0.0
get switch -c v2.0.0
git push
```
