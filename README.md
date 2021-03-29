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

- An environment based on python 3.6 or later
- A `ServiceX` end-point. This is usually gotten by logging into and getting approved at the servicex endpoint. Once you do that, you'll have an API token, which this library needs to access the `ServiceX` endpoint, and the web address where you got that token (the `endpoint` address).

### How to access your endpoint

The API access information is normally placed in a `.servicex` file (to keep this confidential information form accidentally getting checked into a public repository). The `servicex` library searches for configuration information in several locations to determine what end-point it should connect to, in the following order:

1. A `.servicex` file in the current working directory (it can also be named `servicex.yaml` or `servicex.yml`)
1. A `.servicex` file in the user's home directory (`$HOME` on Linux and Mac, and your profile
   directory on Windows).
1. The `config_defaults.yaml` file distributed with the `servicex` package.

If no endpoint is specified, then the library defaults to the developer endpoint, which is `http://localhost:5000` for the web-service API. No passwords are used in this case.

Create a `.servicex` file, in the `yaml` format, in the appropriate place for your work that contains the following (for the `xaod` backend; use `uproot` for the uproot backend):

```yaml
api_endpoints:
  - endpoint: <your-endpoint>
    token: <api-token>
    type: xaod
```

All strings are expanded using python's [os.path.expand](https://docs.python.org/3/library/os.path.html#os.path.expandvars) method - so `$NAME` and `${NAME}` will work to expand existing environment variables.

You can list multiple end points by repeating the block of 4 dictionary items, but using a different type. For example, `uproot`.

Finally, you can create the objects `ServiceXAdaptor` and `MinioAdaptor` by hand in your code, passing them as arguments to `ServiceXDataset` and inject custom endpoints and credentials, avoiding the configuration system. This is probably only useful for advanced users.

## Usage

The following lines will return a `pandas.DataFrame` containing all the jet pT's from an ATLAS xAOD file containing Z->ee Monte Carlo:

```python
    from servicex import ServiceXDataset
    query = "(call ResultTTree (call Select (call SelectMany (call EventDataset (list 'localds:bogus')) (lambda (list e) (call (attr e 'Jets') 'AntiKt4EMTopoJets'))) (lambda (list j) (/ (call (attr j 'pt')) 1000.0))) (list 'JetPt') 'analysis' 'junk.root')"
    dataset = "mc15_13TeV:mc15_13TeV.361106.PowhegPythia8EvtGen_AZNLOCTEQ6L1_Zee.merge.DAOD_STDM3.e3601_s2576_s2132_r6630_r6264_p2363_tid05630052_00"
    ds = ServiceXDataset(dataset)
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

### The Local Cache

To speed things up - especially when you run the same query multiple times, the `servicex` package will cache queries data that comes back from Servicex. You can control where this is stored with the `cache_path` in the `.servicex` file (see below).

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

## Configuration

As mentioned above, the `.servicex` file is read to pull a configuration. The search path for this file:

1. Your current working directory
2. Any working directory above your current working directory.
3. Your home directory

The file can be named any of the following (ordered by precedence):

- `servicex.yaml`
- `servicex.yml`
- `.servicex`

The file can contain an `api_endpoint` as mentioned above. In addition the other following things can be put in:

- `cache_path`: Location where queries, data, and a record of queries are written. This should be an absolute path the person running the library has r/w access to. On windows, make sure to escape `\` - and best to follow standard `yaml` conventions and put the path in quotes - especially if it contains a space. Top level yaml item (don't indent it accidentally!). Defaults to `/tmp/servicex` (with the temp directory as appropriate for your platform) Examples:
  - Windows: `cache_path: "C:\\Users\\gordo\\Desktop\\cacheme"`
  - Linux: `cache_path: "/home/servicex-cache"`

- `backend_types` - a list of yaml dictionaries that contains some defaults for the backends. By default only the `return_data` is there, which for `xaod` is `root` and `uproot` is `parquet`. Allows `servicex` to convert to `pandas.DataFrame` or `awkward` if requested by the user.

All strings are expanded using python's [os.path.expand](https://docs.python.org/3/library/os.path.html#os.path.expandvars) method - so `$NAME` and `${NAME}` will work to expand existing environment variables.

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
- Python 3.6, 3.7, 3.8
- Jupyter Notebooks (not automated), regular python command-line invoked source files

### Non-standard backends

When doing backend development, often ports 9000 and 5000 are forwarded to the local machine exposing the `minio` and `ServiceX_App` instances. In that case, you'll need to create a `.servicex` file that has `http://localhost:5000` as the end point. No API token is necessary if the development `ServiceX` instance doesn't have authorization turned on.

## API

Everything is based around the `ServiceXDataset` object. Below is the documentation for the most common parameters.

```python
  ServiceXDataset(dataset: str,
                 backend_type: Optional[str] = None,
                 image: str = 'sslhep/servicex_func_adl_xaod_transformer:v0.4',
                 max_workers: int = 20,
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
          backend_type                The type of backend. Used only if we need to find an
                                      end-point. If we do not have a `servicex_adaptor` then this
                                      will default to xaod, unless you have any endpoint listed
                                      in your servicex file. It will default to best match there,
                                      in that case.
          image                       Name of transformer image to use to transform the data
          max_workers                 Maximum number of transformers to run simultaneously on
                                      ServiceX.
          servicex_adaptor            Object to control communication with the servicex instance
                                      at a particular ip address with certain login credentials.
                                      Will be configured via the `.servicex` file by default.
          minio_adaptor               Object to control communication with the minio servicex
                                      instance. By default configured with values from the
                                      `.servicex` file.
          cache_adaptor               Runs the caching for data and queries that are sent up and
                                      down.
          status_callback_factory     Factory to create a status notification callback for each
                                      query. One is created per query.
          local_log                   Log adaptor for logging.
          session_generator           If you want to control the `ClientSession` object that
                                      is used for callbacks. Otherwise a single one for all
                                      `servicex` queries is used.
          config_adaptor              Control how configuration options are read from the
                                      `.servicex` file.

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
 |  get_data_awkward_async(self, selection_query: str) -> Dict[bytes, Union[awkward.array.jagged.JaggedArray, numpy.ndarray]]
 |      Fetch query data from ServiceX matching `selection_query` and return it as
 |      dictionary of awkward arrays, an entry for each column. The data is uniquely
 |      ordered (the same query will always return the same order).
 |
 |  get_data_awkward(self, selection_query: str) -> Dict[bytes, Union[awkward.array.jagged.JaggedArray, numpy.ndarray]]
 |      Fetch query data from ServiceX matching `selection_query` and return it as
 |      dictionary of awkward arrays, an entry for each column. The data is uniquely
 |      ordered (the same query will always return the same order).
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
