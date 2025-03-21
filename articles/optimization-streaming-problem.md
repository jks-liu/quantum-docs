---
author: sonialopezbravo
description: Reference for azure.quantum.optimization.StreamingProblem
ms.author: sonialopez
ms.date: 05/01/2023
ms.service: azure-quantum
ms.subservice: optimization
ms.topic: reference
title: Streaming upload of large optimization problems
uid: microsoft.quantum.optimization.streaming-problem
---

# Streaming upload of large optimization problems

[!INCLUDE [QIO deprecation warning](includes/qio-deprecate-warning.md)]

When formulating large problems with the *azure-quantum* Python package, you may find that you do not have enough memory to keep the entire problem definition loaded, which is the behavior of the [`Problem`](xref:microsoft.quantum.optimization.problem) class. If you do not need to keep your whole problem definition in memory for later access or modification you should consider using the `StreamingProblem` class instead, which is a drop-in replacement for the `Problem` class, but that streams the problem definition to Azure as you formulate the problem to reduce memory requirements and increase performance.

## StreamingProblem

```py
from azure.quantum.optimization import StreamingProblem
```

The `StreamingProblem` class supports the same interface for adding terms to a problem definition as the `Problem` class. However, once terms are added to the problem, they are queued to be uploaded by a background thread and are not kept in memory for future reference.

### Lifecycle of a StreamingProblem

The first time you call one of the `add_term()` or `add_terms()` functions on a streaming problem, the following actions happen:

1. A background thread is started.
2. A [Block Blob](/rest/api/storageservices/understanding-block-blobs--append-blobs--and-page-blobs) is initialized in Azure Storage, which is where the problem will be uploaded.
3. The terms that were added are queued for upload.

The background thread remains alive until the problem definition is finalized and continuously attempts to chunk terms into blocks that are individually uploaded in Azure. A chunk upload is triggered when the queued terms surpass a threshold on the number of terms or the size of the data. This means that not every call to `add_terms()` triggers an upload.

After the first call to `add_terms()`, you may continue to call this method as many times as needed to add all of your terms. Because this method is thread-safe, you may have multiple threads simultaneously adding terms to the problem definition although there is only ever one uploader thread.

Once you have finished adding all of your terms, you must call the `upload()` method to finalize the problem definition. (`upload()` is called automatically if you call `solver.submit(problem)`). The first time the `upload()` method is called, the streaming problem will block on the background upload thread to complete. Once this is done, the block blob is finalized in Azure Storage and is ready to be used to solve the problem with Azure Quantum. Be aware that may not add terms to a problem after it has been finalized.

If `upload()` is not called on a streaming problem, the block blob in Azure Storage is never finalized. When a block blob isn't finalized, Azure Storage automatically deletes the uploaded data within a period of time. For more information, see the [Block Blob reference](/rest/api/storageservices/understanding-block-blobs--append-blobs--and-page-blobs).

### Tuning the upload

Depending on the characteristics of your problem (especially density) or of your CPU or network connection, you may want to tune the StreamingProblem class. There are two options for tuning the upload:

- `StreamingProblem.upload_size_threshold` - the size, in bytes, of the compressed payload to upload. As terms are added they are compressed on the fly. When the size of the staged compressed payload surpasses this threshold, the chunk is uploaded.
- `StreamingProblem.upload_terms_threshold` - the threshold for the number of terms that trigger an upload. When the number of queued terms exceeds this threshold, an upload is triggered. If your problem has high connectivity (for example, if your terms have many variables), you can choose to lower this threshold.

To tune these parameters, set one or both as shown in the following example **before adding terms to the problem definition**. Changing these parameters after the background uploader begins has no effect.

```py
problem = StreamingProblem(workspace = workspace, name="My Streamed Problem", problem_type=ProblemType.ising)
problem.upload_size_threshold = 10e6
problem.upload_terms_threshold = 1000
```

In general:

- If you have plenty of free memory, consider _increasing_ the thresholds
- If you have a slow network connection, consider _increasing_ the thresholds
- If you are highly memory constrained, consider _decreasing_ the size threshold

### Statistics of the uploaded problem

The `StreamingProblem` class calculates some statistics about the generated problem as terms are added. To see information about the terms already added to the problem, access the `stats` member of the class, which is a dictionary of calculated statistics and contains the following information:

```py
problem = StreamingProblem(workspace = workspace, name="My Streamed Problem", problem_type=ProblemType.ising)
# Terms are added...
print(problem.stats)

> {'type': 'ising', 'max_coupling': 4, 'avg_coupling': 2.4, 'min_coupling': 2, 'num_terms': 6125}
```

- `type`: The type of problem - `pubo` or `ising`
- `max_coupling`: The largest coupling (number of variable indices) in any term
- `avg_coupling`: The average coupling (number of variable indices) of all terms
- `min_coupling`: The smallest coupling (number of variable indices) in any term
- `num_terms`: The total number of terms

These statistics are also set as metadata on the uploaded blob in Azure Storage. For more information, see the [Azure Storage documentation](/training/modules/organize-blobs-properties-metadata/) for more information on accessing these properties.

## Constructor

To create a `StreamingProblem` object, specify the following information:

- `workspace`: The Azure Quantum workspace that this problem is being uploaded to. The problem will be stored in the linked storage account from the workspace.
- `name`: A friendly name for your problem. The name does not need to be unique.
- (Optional) `terms`: A list of `Term` objects to add to the problem. If provided, this will start the background uploader.
- (Optional) `problem_type`: The type of problem. Must be either `ProblemType.ising` or `ProblemType.pubo`. Default is `ProblemType.ising`.

```py
problem = StreamingProblem(workspace = workspace, name="My Streamed Problem", problem_type=ProblemType.ising)
```

## StreamingProblem.add_term

Adds a single term to the problem. It takes a coefficient for the term and the indices of variables that appear in the term.

Thread-safe: may be called from multiple calling threads.

```py
coefficient = 0.13
problem.add_term(c=coefficient, indices=[2,0])
```

## StreamingProblem.add_terms

Adds multiple terms to the problem using a list of `Terms`.

Thread-safe: may be called from multiple calling threads.

```py
problem.add_terms([
    Term(c=-9, indices=[0]),
    Term(c=-3, indices=[1,0]),
    Term(c=5, indices=[2,0]),
    Term(c=9, indices=[2,1]),
    Term(c=2, indices=[3,0]),
    Term(c=-4, indices=[3,1]),
    Term(c=4, indices=[3,2])
])
```

## StreamingProblem.download

Downloads the uploaded problem definition as an instance of [`Problem`](xref:microsoft.quantum.optimization.problem). May only be called on a streaming problem after it has been finalized by calling `upload()`.

## StreamingProblem.upload

Finalizes the upload of the streaming problem in Azure Storage by blocking on the background uploader thread until it completes. Once this method returns, the problem is ready to be submitted to an Azure Quantum Solver.

This method may be called multiple times. If the upload has not yet completed, all calls will block. If the upload has completed, it immediately returns the URL of the uploaded problem (without re-uploading).

```py
problem.upload()
```
