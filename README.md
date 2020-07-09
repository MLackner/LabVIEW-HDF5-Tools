## Dependencies

* OpenG Toolkit
* UPVI Live HDF5

Somewhere is stated that there may be conflicts between UPVI Live HDF5 and h5labview.
Make sure h5labview is not installed. UPVI Live HDF5 does only work on Windows machines.

## Functionality

To get an overview of what this does take a look at the following Julia code:

```julia
using HDF5

"""
Creates an empty hdf5 file with the correct
dimensions for the datasets. We need the data
dictionary to get the type of the datasets
"""
function createfile(filename::String, data::Dict, dims::Tuple{Int,Int}, mode::String)
    # Open a new file. Destroy existing content.
    h5open(filename, "w") do f
        # Create a new general group (intended for light, dark, flat, darkflat or bias)
        g = creategroup(f, mode)
        # Inside this newly created group we then create a group/dataset structure
        # that resembles the 'data' dictionary. We need the dimensions to preallocate
        # the datasets.
        createstructure(g, data, dims)
    end
end

function createstructure(parent, child::Dict, dims)
    for name in keys(child)
        # loop through the keys of the dictionary
        if child[name] isa Dict
            # In case the current element is a dictonary itself, create a new
            # group and inside that create a group/dataset structure (recursive
            # call)
            g = creategroup(parent, name)
            createstructure(g, child[name], dims)
        else
            # Create an empty dataset with the correct dimensions
            data = child[name]
            createdataset(parent, name, data, dims)
        end
    end
end

function createdataset(parent, name::String, data, dims)
    dtype = eltype(data)
    nativedims = size(data)
    datadims = (nativedims..., dims...)
    chunkdims = (nativedims..., fill(1, length(dims))...)
    d = d_create(parent, name, datatype(dtype), dataspace(datadims), "chunk", chunkdims)
end

function creategroup(parent, groupname::String)
    g = g_create(parent, groupname)
end

function savetodataset(dset, data, indices)
    nativedims = size(data)
    datadims = size(dset)
    native_indices = [1:_d for _d in nativedims]
    dset[native_indices..., indices...] = data
    nothing
end

function walkstructure(parent, child::Dict, indices)
    for name in keys(child)
        if child[name] isa Dict
            # Go inside that group
            walkstructure(parent[name], child[name], indices)
        else
            dset = parent[name]
            data = child[name]
            savetodataset(dset, data, indices)
        end
    end
end

function savedata(filename, data::Dict, indices, mode)
    h5open(filename, "r+") do f
        g = f[mode]
        walkstructure(g, data, indices)
    end
end
```