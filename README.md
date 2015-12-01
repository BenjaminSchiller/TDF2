# TDF2



## Components

![Components of TDF](doc/components.pdf "Components of TDF")

Tdf consists of three components:

1. *server*
2. *worker*
3. *cli*

The **server** is the central component that maintains tasks, task lists, and their respective states.

The **workers** fetch tasks grouped int tasks lists, work on them, and write back the results to the server.

The **cli** is a *command line interface* for creating new tasks on the server, managing the lists stored there, and obtaining information, statistics, and results from there.

## Tasks

A task contains the following fields:

1. **id** - a unique identifier (assumed to be ascending)
1. **created** - timestamp when this task was initially created1. **until**  - timestamp when this task should be started the latest1. **expected_duration** - expected duration for executing this task
1. **max_fails** - maximum number this task is allowed to fail
1. **fails** - number of times this task failed so far1. **cmd** - the actual command that should be executed (THE TASK)
1. [**worker**] - name of the worker that executed this task
1. [**started**] - timestamp when the worker started the execution
1. [**ended**] - timestamp when the worker finished the execution

The **id** is assumed to a unique scending number. For simplicity, we assume it to be the unix timestamp in *ns* followed by an underscore (`_`) and the number of times it already failed, i.e.:

	"$(date +%s%N)_${fails}"

Each timestamp (**created**, **until**, **started**, **ended**) is assumed to be the unix timestamp in *sec*, i.e.:

	$(date +%s)

The **expected_duration** must be given in *sec*.

A task is stored in a file where each field is stored in each line.
The last 3 field (**worker**, **started**, **ended**) are added by the worker.
The following gives an example of a task:

	1448978609795035000_0			# id
	1448978609						# created
	1448985809						# until
	120								# expected_duration
	5								# max_fails
	0								# fails
	echo 'A'; sleep 2; echo 'B'		# cmd

It was created at *Tue, 01 Dec 2015 14:03:29 GMT* (`1448978609`) and should be finished 2 hours later (`1448985809 = 1448978609 + 60*60*2`).
The **id** is the timestamp of the creation in nanoseconds followed by `_0` as it never failed so far (`1448978609795035000 `).
As an upper bound (nonsense in this case) an upper bound for its execution is given with *2 minutes* (`120` sec).
The task should not fail more than `5` times and has not failed yet (`0`).
Finally, the command (as it always needs to be) is a single line (`echo 'A'; sleep 2; echo 'B'`).

Now assume that worker **worker_123** executed this task 10 seconds after its creation and this execution took 3 seconds. Afterwards, the task file would contain the following:

	1448978609795035000				# id
	1448978609						# created
	1448985809						# until
	120								# expected_duration
	5								# max_fails
	0								# fails
	echo 'A'; sleep 2; echo 'B'		# cmd
	worker_123						# worker
	1448978619						# started
	1448978622						# ended

A task (in this format, described above) is stored in a file with name `${id}.task`.
When executing a task at the worker, the following two files exist:

1. `${id}.out`
1. `${id}.err`

During execution, *stdout* is redirected into `${id}.out` and *stderr* is redirected in `${id}.err`.
The content of the first is expected to contains the result of the command execution while the second one should be empty in case no errors occurred during execution.

Hence, a successfully executed task consists the `${id}.task` file with all 10 lines / fields and the result of the execution stored in `${id}.out`.
We assume, that a worker does not transmit `${id'.err` in case it is empty (and even deleted its local copy).
Thereby, a task is flagged as failed (on the server side) by the existance of a non-empty `${id}.err` file.



## Task Lists

A task list is (as complicated as it may appear) a list of tasks.
It is simply represented as a directory containing an arbitrary number of tasks represented by a file each.

A task list can have three states:

1. *open* - `${timestamp}`
2. *running* - `${timestamp}_${worker}_${started}`
3. *closed* - `${timestamp}_${worker}_${started}_${finished}`

An *open* task list is created by combining multiple tasks.
The name of the representing directory is the minimum  creation timestamp of any contained task to ensure the similar ordering of tasks list as for tasks.

When a worker claims an *open* task list, it is moved to the *running* state.
In addition, the name of the worker and the timestamp when it started to work on it are appended.

After a worker finished the execution of a *running* task list, it is moved to the *closed* state and the corresponding files are transferred from the worker to the server.





## States of Tasks and Task Lists

![State transitions](doc/states.pdf "State transitions")

Tasks **[T]** and task lists **[TL]** can have the following states:

1. **[T]** *new*
1. **[TL]** *open*
1. **[TL]** *running*
1. **[TL]**  *closed*
1. **[T]** *completed* - *completed_archive*
1. **[T]** *failed* - *failed\_archive* / *failed\_time* / *failed\_count*

A task is *new* after it was **created**.
Such a *new* task contains only the first 7 fields.

After being *new*, a task is **assigned to a task list** which becomes *open*.

An *open* task list is then fetched by a worker, changing its state to be *running*.

After a worker finished the execution of a task list (i.e., the consecutive execution of all tasks contained therein), the task list is moved to the state *closed*.
In addition to this state transition, the worker adds the last 3 fields to the task file and copies the existing out and err files.

In case a worker crashes and hence cannot finish the execution of a task list, the task list is re-opened, i.e., moved to the state *open* again.
Based on the `expected_duration` field of all tasks contained in a list, we assume the expected execution of a whole task list to be below the sum of the expectation for all tasks.
Hence, a task list is re-opened in case this timeout is hit.

After being processed by a worker as part of a task list, a taskcan be either *completed* (i.e., the execution was successful, `${id}.err` does not exist) or the state *failed* (i.e., the execution has failed, `${id}.err` exists).

A *completed* task is moved to the *completed_archive* state (after potential processing).

A *failed* task should be attempted to be executed again.
This is not done in case it failed too many times or the until timestamp is exceeded.
In case it was already executes too many times, it is moved to the *failed_count*.
In case the until timestamp is exceeded, the task is moved to the state *failed_time*).
If both cases are not met, the task is moved to the *failed_archive* state and a duplicate of the task is created in the state *new*.
For this duplicate, the `fails`counter is increased and stored in the field, the id, and the filename.


