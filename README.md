simple_gpu_scheduler
--------------------

A simple scheduler to run your commands on individual GPUs.  Following the
[KISS principle](https://en.wikipedia.org/wiki/KISS_principle), this script
simply accepts commands via `stdin` and executes them on a specific GPU by
setting the `CUDA_VISIBLE_DEVICES` variable.

The commands read are executed using the login shell, thus redirections `>`
pipes `|` and all other kinds of bash magic can be used.

Installation
------------

The package can simply be installed from
[pypi](https://pypi.org/project/simple-gpu-scheduler/)
```bash
$ pip install simple_gpu_scheduler
```

Example
-------

To show how this generally works, we will create jobs that simply outputs
a job id and the value of `CUDA_VISIBLE_DEVICES`:

```bash
for i in {0..10}; do echo "echo job_id=$i device=\$CUDA_VISIBLE_DEVICES && sleep 3"; done | simple_gpu_scheduler --gpus 0,1,2
```

which results in the following output:

```
Processing `command echo job_id=0 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 2
Processing `command echo job_id=1 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 1
Processing `command echo job_id=2 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 0
job_id=0 device=2
job_id=1 device=1
job_id=2 device=0
--- 3 seconds no output ---
Processing command `echo job_id=3 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 2
Processing command `echo job_id=4 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 1
Processing command `echo job_id=5 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 0
job_id=3 device=2
job_id=4 device=1
job_id=5 device=0
--- 3 seconds no output ---
Processing command `echo job_id=6 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 2
Processing command `echo job_id=7 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 1
Processing command `echo job_id=8 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 0
job_id=6 device=2
job_id=7 device=1
job_id=8 device=0
--- 3 seconds no output ---
Processing command `echo job_id=9 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 2
Processing command `echo job_id=10 device=$CUDA_VISIBLE_DEVICES && sleep 3` on gpu 0
job_id=9 device=2
job_id=10 device=0
```

This is equivalent to creating a file `commands.txt` with the following content:

```bash
echo job_id=0 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=1 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=2 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=3 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=4 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=5 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=6 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=7 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=8 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=9 device=$CUDA_VISIBLE_DEVICES && sleep 3
echo job_id=10 device=$CUDA_VISIBLE_DEVICES && sleep 3
```

and running
```bash
simple_gpu_scheduler --gpus 0,1,2 < commands.txt
```

Simple scheduler for jobs
-------------------------

Combined with some basic command line tools, one can set up a very basic
scheduler which waits for new jobs to be "submitted" and executes them in order
of submission.

Setup and start scheduler in background or in a separate permanent session
(using for example `tmux`):
```bash
touch gpu.queue
tail -f -n 0 gpu.queue | simple_gpu_scheduler --gpus 0,1,2
```
the command `tail -f -n 0` follows the end of the gpu.queue file. Thus if there
was anything written into `gpu.queue` prior to the execution of the command it
will not be passed to `simple_gpu_scheduler`.

Then submitting commands boils down to appending text to the `gpu.queue` file:

```bash
echo "my_command_with | and stuff > logfile" >> gpu.queue
```

Hyperparameter search
---------------------

In order to allow user friendly utilization of the scheduler in the common
scenario of hyperparameter search, a convenience script `simple_hypersearch` is
included in the package.

```bash
simple_hypersearch -h
```

```bash
usage: simple_hypersearch [-h] [--sampling-mode {shuffled_grid,grid}]
                          [--n-samples N_SAMPLES] [--seed SEED]
                          [-p NAME [VALUES ...]]
                          command_pattern

Convenience tool to generate hyperparameter search commands from a command pattern and parameter ranges.

positional arguments:
  command_pattern       Command pattern where placeholders with {parameter_name} should be replaced.

optional arguments:
  -h, --help            show this help message and exit
  --sampling-mode {shuffled_grid,grid}
                        Determine how to sample commands. Either in the grid order [grid]
                        or in a shuffled order [shuffled_grid, default].
  --n-samples N_SAMPLES
                        Number of samples to draw. If not provided use all possible combinations.
  --seed SEED           Random seed to ensure reproducability when using randomized order of the grid.
  -p NAME [VALUES ...], --parameter NAME [VALUES ...]
                        Name of parameter followed by values that should be considered for hyperparameter search.
                        Example: `-p lr 0.01 0.001 0.0001`

Usage example:
    simple_hypersearch "my_program --param1 {param1} --param2 {param2}" -p param1 0 1 -p param2 2 3
    will generate the output:
    my_program --param1 0 --param2 2
    my_program --param1 0 --param2 3
    my_program --param1 1 --param2 2
    my_program --param1 1 --param2 3
```

This allows to easily perform hyperparameter searches over a grid of values or
uniform samples of the grid (dependent on the setting of `sampling-mode`).
The output can directly be piped into `simple_gpu_scheduler` or appended to the
"queue file" (see [Simple scheduler for jobs](#simple-scheduler-for-jobs)).

Here some more concrete examples:

**Grid of all possible parameter configurations in random order:**
```bash
simple_hypersearch "python3 train_dnn.py --lr {lr} --batch_size {bs}" -p lr 0.001 0.0005 0.0001 -p bs 32 64 128 | simple_gpu_scheduler --gpus 0,1,2
```

**5 uniformly sampled parameter configurations:**
```bash
simple_hypersearch "python3 train_dnn.py --lr {lr} --batch_size {bs}" --n-samples 5 -p lr 0.001 0.0005 0.0001 -p bs 32 64 128 | simple_gpu_scheduler --gpus 0,1,2
```

TODO
----

 - Multi line jobs (evtl. we would then need a submission script after all)
 - Stop, but let commands finish when receiving a defined signal
 - Tests would be nice, until now the project is still __very small__ but if it
   grows tests should be added
