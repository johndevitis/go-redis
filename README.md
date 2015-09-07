# go-redis

go-redis - **G**NU **O**ctave **redis** client

...but Matlab is supported too. When I don't support Matlab, someone else will do and I have to emulate it with Octave.

Tested with Linux and Mac OS X.

For more detailed information next to this `README.md`, take a look at the Wiki

* [Data-Structure](https://github.com/markuman/go-redis/wiki/Data-Structure)
* [Collaborative Workspace](https://github.com/markuman/go-redis/wiki/Collaborative-Workspace)
* [Gaussian Elimination](https://github.com/markuman/go-redis/wiki/Gaussian-elimination)


# Requirements

* Classdef support
  * Octave >= 4.0 
  * Matlab >= R2012b? _(dunno when classdef was introduced...)_
* C-Compiler
* [hiredis library](https://github.com/redis/hiredis/)
* Linux or Mac OS X _(never tried with Windows)_


# Build instructions

1. install a C compiler
  * Ubuntu: `sudo apt-get install build-essential`
  * Arch:   `sudo pacman -S base-devel`
  * Mac:    Install xcode
2. install [hiredis library](https://github.com/redis/hiredis/)
  * Ubuntu: `sudo apt-get install libhiredis-dev libhiredis0.10` _*for 14.04 LTS_
  * Arch:   `sudo pacman -S hiredis`
  * Mac:    `brew install hiredis`
3. mex.h
  * Distributed with your Matlab or GNU Octave installation
4. clone/download and build go-redis directly from Matlab/GNU Octave

    >> cd go-redis/mex
    >> setup
    >> % go where ever you want and just do "addpath ('go-redis/inst')"

### Manually Matlab Instruction

You can compile it directly in the Matlab commandline.

    mex -lhiredis -I/usr/include/hiredis/ CFLAGS='-Wall -Wextra -fPIC -std=c99 -O2 -pedantic -g' redis_.c

Afterwards mv `redis_.mex*` from `mex` folder into `inst/private` folder.

### Manually GNU Octave Instruction

From GNU Octave commandline

    mkoctfile -Wall -Wextra -v -I/usr/include/hiredis --mex redis_.c -lhiredis -std=c99

Afterwards mv `redis_.mex` from `mex` folder into `inst/private` folder.

**Currently (3/19/2015) there is a bug in classdef. You have to do `addpath private` in octave as a workaround!**
https://savannah.gnu.org/bugs/?41723

### Manually Bash Instruction

You can compile it in bash too

    gcc -fPIC -I <PATH TO mex.h> -lm -I <PATH TO hiredis.h> -lhiredis -std=c99 -shared -O2 redis_.c -o redis_.mex

e.g. 

    gcc -fPIC -I /usr/include/octave-4.0.0/octave/ -lm -I /usr/include/hiredis/ -lhiredis -std=c99 -shared -O2 redis_.c -o redis_.mex


# todo

* maybe add `hiredis` as a submodule to simplify the setup process?
* improve c-code
* still some problems with unicodes between matlab and GNU Octave
* improve `pipeline` (subclass, whitespace-safe, etc.)

# limitations

* GNU Octave and Matlab
  * `sscan`/nested cells as return are not supported yet

* GNU Octave
  * there is a bug in classdef. You have to do `addpath private` in octave as a workaround! https://savannah.gnu.org/bugs/?41723
  * `inputname` is currently not supported in a classdef environment. So you have to name you array by yourself when using `array2redis`.



## basics

### initialize redis class

	>> help redis
	 redis mex client for Matlab and GNU Octave
  	r = redis()
  	r = redis(hostname)
  	r = redis(hostname, port)
  	r = redis(hostname, port, db)
  	r = redis(hostname, port, db, pwd)

### properties

 * `hostname`
   * type char
   * default: `127.0.0.1`
 * `port`
   * type double
   * default: `6379`
 * `db`
   * type double
   * default: `0`
   * database number to use
 * `pwd`
   * type char
   * default: empty
   * auth password
 * `precision`
   * type double
   * default: `4`
   * number of decimal points stored in `array2redis()`
 * `batchsize`
   * type double
   * default: `64`
   * when number of commands in `pipeline` == `batchsize`, it automatically execute the `pipeline`.

## usage

### make a connection

    >> r = redis()

	r =

           hostname: '127.0.0.1'
               port: 6379
                 db: 0
             passwd: ''
          precision: 4
          batchsize: 64

### ping the redis server

        ret = r.ping

        ret =

        PONG

### SET, GET, GETSET, APPEND
`r.set(key, value)`  
`r.get(key)`  
`r.getset(key, value)`  
`r.append(key, value)`  

 * value can be a double or a char (expect `append` there it has to be a char)
 * doubles will be converted to char

        ret = r.set('go-redis', 1)

        ret =

        OK

  * return type of `GET*` commands will be a char or a double _(depends on the reply of hiredis)_

### INCR, DECR
`r.incr(key)`  
`r.decr(key)`

* return type will be double.

        ret = r.incr('go-redis')

        ret = 2



### DEL
`r.del(key)`

* return will be true (1) or false (0)

        >> r.del('s')

        ans =

             1

### MOVE
`r.del(key, dbnr)`

* return will be true (1) or false (0)
* `dbnr` can be a char `'1'` or a double `1`. 

### RENAME
`r.rename(oldkeyname, newkeyname)`

* return will be `OK` or `ERR: ...` 
* keynames have to be chars

### SAVE
`r.save()`

* return will be `OK`

### TYPE
`r.type(key)`
return will be a string

        >> r.type('s')

        ans =

             string

### array reply
An array reply will be transformed into a cell array in Octave/Matlab.

        octave:2> r.call('keys *')
        ans =
        {
          [1,1] = b
          [2,1] = A
        }


### CALL
`r.call(command)`

* for debugging 
* functions that are not directly supported by redis() class
* for disable the overhead of redis() class functions
* command can be a string or a cell (e.g. `{'SET', 'my keyname', 'this is a value'}`)

### PIPELINE
`r.pipeline(command)` 
`r.execute()` 

Using `r.pipeline` will speedup your writing commands 2-3x.

_But be aware, `pipeline()` is not implemented as a subclass! That means you have to put everything into a string by yourself at the moment._

But the cool thing is: the pipeline is executed automatically when a number of commands is reached. The default value of `r.batchsize` is `64`.
So you just need to call `r.execute()` (Yes, it takes no arguments!) one time when you're done to get execute the rest in your pipeline.

#### pipeline examples
    r = redis();
    r.call('SET M 0');
    tic
    for n = 1:5000
        r.call('INCR M');
    end
    toc
    
    tic
    for n = 1:5000
        r = r.pipeline('INCR M');
    end
    r = r.execute();
    toc
    
    tic
    for n = 1:5000
        r.call('SET M 5');
    end
    toc
    
    tic
    for n = 1:5000
        r = r.pipeline('SET M 5');
    end
    r = r.execute();
    toc


But you can pass a cell array of arguments too, to bypass the class functionality and its magic pipe execution.

    r.call({'SET A 0'; 'INCR A'; 'INCR A'})

### GNU OCtave and Matlab special

#### array2redis
`r.array2redis(array, name)`
For storing a multidimension numeric array in redis

        >> r.array2redis(rand(3,3), 'm')

        ans =

             1

#### redis2array
`r.redis2array(name)`
For reading a multidimension numeric array from redis back into workspace

        >> r.redis2array('m')

        ans =

            0.8147    0.9134    0.2785
            0.9058    0.6324    0.5469
            0.1270    0.0975    0.9575



# deprecated go-redis version

For the older go-redis version - pure written in Octave using
[instrumen-control](http://octave.sourceforge.net/instrument-control/index.html) package - do `git checkout fcf757b`



