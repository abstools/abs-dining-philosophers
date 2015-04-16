
The Dining Philosophers Problem {#chap:philosophers}
===============================

This chapter gives an introduction to the Dining Philosophers Problem and
shows how it can be implemented in ABS. The problems of deadlock and
starvation are explained, and it is shown how they appear in the Dining
Philosophers. Finally, some solutions to the problem are discussed, and Chandy 
and Misra's *hygienic solution* is explained and implemented.

Problem outline <!-- ?? -->
---------------

The Dining Philosophers is a well-known problem in the world of concurrent 
programming, formulated by Edsger W. Dijkstra in 1965. Originally titled *The 
problem of The Dining Quintuple*, it was first used as an examination problem 
in a course Dijkstra was teaching [@ewd1000]. <!-- ... at XXX university --> 

Serving as a metaphor on computers running in a network with common resources 
such as peripherals that require exclusive access, the dining philosophers 
problem efficiently demonstrates how a deadlock, or "deadly embrace", may 
occur.

The scenario involves 5 philosophers who sit around a table. All they do is 
alternate between thinking and eating. Between each pair of adjacent 
philosophers there is one fork on the table. For some reason the philosophers 
need two forks in order to eat, and they can only use the forks that lie next 
to them on the table.

At most one philosopher can hold the same fork at a given point of time, so 
clearly, since a philosopher shares forks with the ones on either side of him, 
two philosophers sitting next to each other can not eat simultaneously 
[@ewd310].

<!-- TODO: better placement of citation? --->

Dijkstra provides the following pseudo-code outline of a naive implementation 
of the philosophers, using a semaphore for each of the forks as a way of 
picking them up. After acquiring the semaphores that correspond to his forks, a 
philosopher may eat without interruption until he releases the semaphores 
again.

\begin{verbatim}
cycle begin think;
            P(left hand fork);
            P(right hand fork);
            eat;
            V(left hand fork);
            V(right hand fork);
      end
\end{verbatim}


<!-- yolo -->

\begin{figure}
    \includegraphics[width=\linewidth]{figures/philosophers.eps}
    \captionsetup{skip=1.5cm}
    \caption[The dining philosophers]
        {Five Dining Philosophers seated around a table with five forks. Each 
        fork is shared between two adjacent philosophers.}
\end{figure}



Naive ABS implementation {#sec:philimpl}
------------------------

To model the dining philosophers in ABS, semaphores cannot be used, as the 
language does not offer them. Instead, the object-oriented nature of the 
language may be utilized by letting the forks be objects, offering their own 
methods for obtaining and releasing exclusive access. The implementation will 
accordingly need two classes: `Philosopher` and `Fork`, which will also have 
their own interfaces. 

The philosopher objects will simply run an infinite loop grabbing and releasing 
forks. Therefore they don't need any visible methods, and the `Philosopher` 
interface may be left empty. For the forks, two methods are specified, 
corresponding to the two semaphore operations, `P` and `V`. In typical 
object-oriented fashion, these methods are given more informative names, 
respectively `grab` and `release`. When calling either of these methods, a 
philosopher will need to identify itself by passing itself as a parameter.

    interface Philosopher { }

    interface Fork {
        Unit grab(Philosopher owner);
        Unit release(Philosopher owner);
    }

The basic structure of the philosopher class is implemented as follows. The two 
fork objects accessible to a philosopher are passed as class parameters. All 
the logic is put in the `run` method, making the philosopher class active. Each 
time after calling `grab()` on a fork, the `get` statement is used without a 
preceding `await` statement, thus blocking the COG and effectively making the 
calls synchronous.

<!-- TODO update with `await f1!grab()` -->

~~~{#lst:naivep style=numbered caption="Basic implementation of a dining philosopher"}
class Philosopher(Fork left, Fork right)
implements Philosopher {

    Unit run() {
        while (True) {
            // think

            Fut<Unit> f;
            f = left!grab(this);
            f.get;
            f = right!grab(this);
            f.get;
            // eat

            left!release(this);
            right!release(this);
} } }
~~~

It is essential that at any given time, at most one philosopher can hold the 
same fork. It makes sense to leave it to the implementation of the forks to 
ensure that this invariant is enforced. Conveniently, the ABS language's 
cooperative scheduling guarantees that each COG only will have one task running 
at a time. Therefore, critical sections of the program are guaranteed by the 
language semantics.

What must be enforced however, is that only the last philosopher who has 
successfully called `grab()` will be given access to other methods on the fork 
object, even after the call to `grab` has terminated. A field `owner` is 
sufficient; The owner is set with the `grab` method, and reset with 
`release()`. When trying to grab a fork, the philosophers must wait until 
`owner` has been reset. 

If the value of `owner` was not really checked, other than whether or not its 
value is `null`, this would have the same effect as a binary semaphore. The 
field `owner` could then have been replaced by a boolean value indicating 
whether or not the fork was currently being held by a philosopher. However, 
`owner` arguably states the purpose of the variable more clearly. It also makes 
it possible to check that the philosopher who is releasing a fork is in fact 
the current owner.

~~~{#lst:naivefork caption="The fork class"}
class Fork implements Fork {
    Philosopher owner;

    Unit grab(Philosopher p) {
        await owner == null;
        owner = p;
    }

    Unit release(Philosopher p) {
        if (p == owner)
            owner = null;
    }
}
~~~


In order to start the simulation, five `fork`s and five `philosopher`s are 
initialized in the program's main block. All of these objects need to run in 
parallel and independently of each other, so each object is created within its 
own COG.

    {   // Main block:
        Fork f1 = new Fork();
        Fork f2 = new Fork();
        Fork f3 = new Fork();
        Fork f4 = new Fork();
        Fork f5 = new Fork();
        new Philosopher(f1, f2);
        new Philosopher(f2, f3);
        new Philosopher(f3, f4);
        new Philosopher(f4, f5);
        new Philosopher(f5, f1);
    }


Running the basic implementation of the dining philosophers from the 
command-line is not very rewarding; The program merely runs 5 simultaneous, 
infinite loops, but not much appears to happen. The philosophers' main loop in 
\Cref{lst:naivep} contains two comments, on lines 6 and 13. These comments are 
placeholders for some imaginary `think` and `eat` routines respectively.

To show what is actually happening while the program is running, the `run` 
method can be augmented with some code that prints status messages to standard 
output. To tell the output from the different philosophers apart, each message 
should be prefixed with a string identifying the philosopher. For this purpose 
the built-in function `toString()` can be used with the philosopher object as 
parameter.

For code reuse, and to keep non-essential logic out of the `run` method, an 
auxiliary method `tell()` will be defined in the `Philosopher` class. The 
`tell` method will take a string and prefix it with the string representation 
of the philosopher object itself, before passing it to the built-in function 
`println()`.

~~~{caption="A simple method for printing status messages"}
    Unit tell(String s) {
        Unit p = println(toString(this)+" "+s);
    }
~~~

With the `tell` method in place, the placeholder comments can now be replaced 
with calls to `tell()` with appropriate messages such as `"is thinking"`, and 
`"is eathing"`. To show more details of the philosopher's progress, status 
messages can also be printed after each time the philosopher grabs a fork and 
after he releases both forks. The output from a single, isolated philosopher 
process would look like the following, repeated over and over:

\begin{verbatim}
Philosopher 1 is thinking
Philosopher 1 grabbed left fork
Philosopher 1 grabbed right fork
Philosopher 1 is eating
Philosopher 1 released both forks
\end{verbatim}

The full ABS implementation of the Dining Philosophers is listed on 
\vpageref{dpcomplete} in the appendix.


Deadlock
--------

Because all the philosophers grab their respective forks in the same order and 
one at a time, a deadlock is likely to happen. As @ewd310 [p.15] explains, all 
philosophers may get hungry and try to grab their left-hand forks 
simultaneously. If they all operate roughly in the same speed, so that each 
philosopher successfully acquire his left-hand fork, then every philosopher 
will be waiting for his right-hand fork to be available. They are thereby all 
stuck in a cycle.

This can easily be demonstrated with the naive ABS implementation. No 
philosophers are programmed to give up a fork before they have picked up both 
of them, so they are all stuck waiting for each other. Running the full ABS 
program will -- eventually -- show that no philosophers get past the *thinking* 
state and the program stalls. The following is the output from a sample run of 
the ABS implementation, with status messages, where the system enters a 
deadlock.

\begin{verbatim}
Philosopher 4 is thinking
Philosopher 5 is thinking
Philosopher 3 is thinking
Philosopher 1 is thinking
Philosopher 2 is thinking
Philosopher 5 grabbed left fork
Philosopher 2 grabbed left fork
Philosopher 4 grabbed left fork
Philosopher 3 grabbed left fork
Philosopher 1 grabbed left fork
\end{verbatim}



Starvation
----------

As the philosophers run in a loop, they go into the thinking state directly 
after putting down the forks. If one philosopher is waiting for another to put 
down his fork, he is also depending on the scheduler to give him processing 
time at the right moment. The philosopher currently holding said fork could be 
a "fast thinker", meaning that he finishes thinking and has time to pick up the 
forks again within the time given to him by the scheduler.

<!-- TODO ref -->
Even in the ABS implementation there is no guarantee that waiting tasks will be 
given access at the right time. Below is outlined a course of events, showing 
two philosophers `A` and `B` competing for a fork `F`.

\begin{figure*}[h]
\label{starvex}
\begin{verbatim}
Philosopher A calls F!grab();
Philosopher B calls F!grab();      // B's task must wait
Philosopher A calls F!release();
Philosopher A calls F!grab();
\end{verbatim}
\end{figure*}

Intuitively one could assume that after `A` releases the fork, `B`'s `grab` 
would always be granted before `A` manages to sneak in yet another `grab`. This 
does not need to be the case though; ABS will rely on the scheduling of the 
running backend, and philosopher `B` could be left starving.

This type of starvation is arguably more of a problem in a setting where the 
philosophers are local processes running on a single computer, all being 
handled by the same scheduler; and the forks represent some sort of local 
resource which require exclusive access, such as a `volatile` variable or a 
`synchronized` code block in Java. In such a setting, the act of grabbing a 
fork would be entirely performed by the philosophers, while the forks are not 
actually active processes that *do* anything.

  <!-- [^javasync]: TODO brief explanation? Citation? -->

In the ABS model however, the forks are active objects, each running in a 
separate COG. Logically these are to be regarded as entities independent of the 
rest of the system, and they actively react to the incoming requests. It is 
likely that a separate system handling remote procedure calls -- which is what 
the COGs represent -- has a scheduler, or internal queuing system, that would 
try to handle incoming messages roughly in the same order as they arrived.


\begin{figure}[t]
    \stretchgraphics[center]{2cm}{figures/starvation-cropped.pdf}
    \caption
    {Sequence diagram illustrating the starvation of a process}
    \label{fig:starv}
\end{figure}


Still, the basic principle of starvation can be demonstrated with the ABS 
model. \Vref{fig:starv} shows a sequence diagram which is generated from an 
altered version of the implementation of the Dining Philosophers. For clarity 
only two philosophers were initialized in the main block, both with the same 
forks as arguments, and in the same order. The leftmost fork in the diagram is 
the one that the philosophers are competing for, corresponding to fork `A` in 
the example that was outlined in the example \vpageref{starvex}.

The sequence diagram shows the first philosopher successfully grabbing both the 
forks. This can be told from the arrows leading back from the fork processes, 
and the labels telling that the `future` has been resolved. This means that the 
`grab` tasks have terminated and returned to the philosopher. When a `future` 
has been resolved, the `get` statement in the `Philosopher` class no longer 
block the COG, and the philosopher can proceed.

After the first philosopher has acquired the forks, the second philosopher 
issues a `grab` request to the first fork. This request cannot return at this 
time because the first philosopher already has grabbed it. But after 
Philosopher 1 releases both the forks, it immediately sends out new `grab` 
requests. The request to the first fork then competes with the existing request 
from Philosopher 2. In this example Philosopher 1 is granted the forks *again*, 
and we see that Philosopher 2 is still unable to advance.


Solutions
---------

### Anti-symmetry

It has been covered that when the deadlock appears, all the philosophers are 
waiting in a cycle; They are all programmed the same way and they are all in 
the same state (waiting for fork number two). @Lehmann1981 refer to this as 
*symmetry*.

They argue that when the initial configuration is symmetric, the scheduler may 
choose to advance each philosopher one step at the time in a circular order. 
After the scheduler has finished one round, the configuration is again 
symmetric. This is part of their proof to the following theorem
[-@Lehmann1981 p.135]:

> There is no deterministic, deadlock-free, truly distributed and symmetric 
solution to the dinings philosophers problem.

@andrews2000 [p.166] suggests simply letting one philosopher pick up the 
right-hand fork first. This breaks the symmetry and eliminates the possibility 
of circular waiting, as the right-handed philosopher will have to compete with 
his right-hand side neighbor for the *first* fork. One of these two 
philosophers will have to wait, and while that philosopher is waiting he will 
not pick up his *second* fork, which in the meantime will be available for the 
opposite neighbor.

Instead of a separate implementation, letting one philosopher pick up right 
fork first is easily achieved in the ABS model by flipping the order of fork 
arguments as the last philosopher is created. The program could then 
potentially run forever without stalling. There is however no guarantee against 
starvation; Each philosopher is still competing with its neighbors for access 
to the forks.

### Servant process

Another solution, which at first glance may seem like the most obvious one, is 
to declare that the philosophers must grab both the forks at the same time, 
instead of one by one. This would prevent the philosophers to get stuck with 
one fork in the left hand, waiting for the right-hand fork to become available.

In the current setup, however, the philosophers have no means to grab two forks 
in one atomic operation. The forks are running is separate COGs, independent of 
each other. As the COGs communicate through *asynchronous* message calls, 
clearly an atomic operation like that would involve some sort of 
synchronization.

The simplest form of synchronization is arguably to have the forks handled by 
another process -- a servant -- which will grant a philosopher access to both 
his forks simultaneously. The philosophers would then communicate only with 
this servant process, rather than with the forks directly.

In order to prevent that the servant faces the exact same problems as the 
philosophers do on their own, the system should have only one servant which 
handles all the forks. As its task is to make sure that two forks are handed to 
a philosopher simultaneously, it must leave a philosopher waiting if one of the 
forks the philosophers is requesting is not currently available.

The servant solution prevents deadlocks in the system, at the cost of being 
centralized. All the communication goes through the same servant, as opposed to 
the original scheme where the communication was spread evenly among the 
different philosophers and forks.



<!--
The drawback of the servant solution is that all philosophers must communicate 
with the same, centralized process. Even though it

When taking a better look on the ABS implementation of the philosopher in 
\Vref{lst:naivep}, it is however not immediately clear how it would manage to 
obtain both the forks at the same time without any external aid. One attempt at 
this could be to remove the first `get` statement (line 10) and change the 
second one (line 12) into a joint scheduling point awaiting both forks.

The problem with this is that the forks are running in separate COGs -- They do 
not communicate with each other.

Even if one of the `grab` requests is successful, the

could very well

This is a well-know solution
and the philosopher have no control over the order in which these grabs are 
executed.



### TODO in this section

* Dijkstra's thoughts on double `P`-operation?
* Something about centralized solutions -- waiter etc.
    - centralized is the only scheme that can prevent a situation where only 
      one philosopher is eating and the rest are stuck waiting(?!).
    * Only way to realize a double P, as the two forks are independent and 
      can't otherwise be `P`'ed simultaneously.


-->

Chandy and Misra's "hygienic solution"
--------------------------------------

A more complete and fully decentralized solution to the dining philosophers, 
that also fits nicely with the cooperative scheduling of the ABS language, is 
*A hygienic solution to the diners problem* by @Chandy1984. This section gives 
a brief explanation of the solution and walks the reader through an ABS 
implementation.

### Solution overview

Chandy and Misra's solution differs from the original problem by Dijkstra in 
that no fork ever lies on the table, but is always being held by some 
philosopher. When a philosopher wants a fork that he is not currently holding, 
he can therefore not simply pick it up -- He must instead send a *request* to 
the current holder of that fork.

To ensure that the forks are not simply sent back and forth between two 
philosophers, so that none of them have time to use it, the forks will be 
marked as `clean` or `dirty`: When a philosopher hands over a fork to a 
neighbor, he will clean it first. After a philosopher has finished eating, the 
forks he is holding will be marked as dirty. The key point is that a 
philosopher will only grant a request for a fork if the fork is dirty (and he
is not currently eating). This guarantees that a philosopher will always get to
eat at least once before giving up his forks, so the problem of starvation is 
eliminated.

The configuration is initially set up so that one of the philosophers is 
holding two forks while his right-hand neighbor is holding none. The rest are 
holding one fork each in their left hand. For each fork shared between two 
philosophers, the philosopher not currently holding the fork will instead hold 
a *request token* for that fork.

A philosopher sends a request token to another philosopher to indicate that he 
wishes to use the fork and that he wants the other philosopher to send the fork 
to him. When one philosopher is holding both a fork and the request token for 
that fork, then the other philosopher has an outstanding request for that fork 
[@Chandy1984 p.638].

### ABS implementation overview {#subsec:chandyimp}

<!--
The Chandy/Misra solution is more complicated than the naive solution and will 
require more code... (**The rest in the appendix?**) something something. This 
implementation will be realized with the use of... bla bla.
-->

Two of the first problems that have to be considered when implementing the 
Chandy/Misra solution is how to realize the forks and request tokens, and how 
these should be sent between philosophers.

The latter almost solves itself; The only way two ABS objects may exchange 
information across COGs is by asynchronous method calls. The most obvious way 
to realize the sending of request tokens and forks is therefore to have a 
request token sent by an asynchronous method call that -- while the philosopher 
holding it is not eating -- returns the fork upon termination.

An outstanding request for a fork will then occur in the ABS model as a method 
call that has not yet terminated. This means that one philosopher will not be 
holding both a fork and its request token at the same time, which is something 
that occurs in Chandy and Misra's original solution.

Still, two neighboring philosophers will have different relationships to the 
fork they share between them: One philosopher is holding the fork while the 
other is holding a request token for the fork. This prompts <!--XXX "impel"; 
"force" --> the philosophers to keep two pieces of data for each fork; They 
must know whether they are currently holding the fork in question, and they 
must remember the fork's ID so that they can request the correct fork from 
their neighbor. This problem is minimized by a few simplifications in the 
model.

### Forks as an algebraic data type
<!-- XXX difficult past conditional tense -->
<!-- XXX actually, skip this?
In many programming languages, each philosopher would have needed to keep two 
variables for each fork, due to this twofold relationship between forks and 
philosophers: One variable that either pointed to the actual fork or `null`, 
depending on whether or not the philosopher was holding the fork; The other 
variable to remember the fork's ID, so that the philosopher could have 
requested the correct fork from his neighbor. Alternatively it would have been 
possible to keep one permanent reference to the fork and one boolean variable 
indicating whether or not the philosopher was the one currently holding the 
fork in question. 

Having two variables to keep track of a single object seems awkward and is not 
strictly in the spirit of object-orientation. It is arguably be more elegant to 
capsulize these variables in a single class. In this implementation though, the 
forks will not be realized with any classes at all, but rather as an algebraic 
data type.
-->

In this implementation, the forks will be realized as an algebraic data type. 
The main reason for this is that the philosophers will not actually *use* the 
forks for anything, except as tokens indicating that they may start their `eat` 
routine. Neither is there any logic that needs to be placed in the forks -- the 
philosophers are the ones handling everything.

All that the philosophers really need to know is whether they are currently 
holding both forks so they can start to start to eat, and whether a fork is 
clean or dirty when they receive a request for it. This information will be 
stored in the forks. The forks are thereby nothing more than containers of 
data, which is best represented in ABS as algebraic data types.

It has been covered that one philosopher will not be holding both a fork and 
its request token simultaneously. Moreover, the actual request for a fork will 
be done through a asynchronous call to a method defined specifically for this 
purpose. The only real use left for the request token is to indicate that the 
philosopher is *not* holding a fork, analogue to a `null` pointer for objects.

The request token is then, from a single philosopher's point of view, merely 
another state of the fork, along with `clean` and `dirty`. These three values 
can thereby be merged into a single data type, which will be called `FState`. 

    data FState = Clean | Dirty | RequestToken;


With this simplified representation, and because algebraic data types are 
immutable, there is no coupling between a request token and one unique fork 
object. There must be a way to tell the forks apart, and to identify them when 
sending a fork request. This can be achieved by giving the forks an identifying 
integer argument. For clarity a `ForkID` type synonym is declared.

    type ForkID = Int;
    data Fork = Fork(ForkID forkID, FState state);


With the algebraic data type `Fork`, philosopher processes can now easily pass 
information about the forks among each other. The immutability of the three 
different states, as well as the fork IDs, make reasoning simple and clear. If 
for instance a philosopher is holding fork 4, and fork 4 is clean, he will 
store the information about this fork as the expression `Fork(4,Clean)`. The 
neighbor he is sharing fork 4 with will at the same time know it as 
`Fork(4,RequestToken)`.


### `Philosopher` class overview

<!-- TODO
     on method calls instead of message passing:
     in the current scheme the philosophers do not check whether or not they 
     have an outstanding request for a fork while they think or before they 
     start to eat. Instead The implementation rely on the scheduler: `suspend` 
     and `await` are called, which leaves a gap for the request to be given 
     processor time -->

Unlike in the naive solution, the philosophers can not start running right upon 
their creation. They all need to know about their neighbors so that they can 
pass request tokens and forks to each other. As the neighbor relationship is 
circular, all the philosophers must be initialized before each is given 
references to his neighbors. A method `start` will be used to pass information
about a philosopher's neighbors and indicate that the main routine may begin.

As has been covered, these requests will be done through asynchronous method 
calls. The method `request` takes a `ForkId` argument and returns a `Fork`.

    interface Philosopher {
        Unit start(Philosopher left, Philosopher right);
        Fork request(ForkID i);
    }

The forks, on the other hand, can be passed directly to the philosophers as the 
philosophers are created, just like in the implementation of the naive solution 
Either way, there is no *initialization* of an algebraic data type. For brevity 
the left and right forks are simply called `forkL` and `forkR` respectively, 
and the fields holding the philosopher's neighbors are similarly called `philL` 
and `philR`.

~~~{#lst:chandyp caption="Beginning of the Chandy/Misra Philosopher class"}
class Philosopher(Fork forkL, Fork forkR)
implements Philosopher {
    Philosopher philL;
    Philosopher philR;

    Unit start(Philosopher left, Philosopher right) {
        philL = left;
        philR = right;
~~~

Once its main routine begins, a philosopher will cycle trough the states 
*thinking*, *hungry* (waiting for one or two forks) and *eating*. Except for 
when in the eating state, a philosopher should be able to handle incoming fork 
requests. But, with the cooperative scheduling in ABS, a fork request will not 
run in parallel with the philosopher's thinking routine. Instead, explicit 
scheduling points must be inserted at convenient locations in the code.


### Scheduling points

If the philosopher was a part of an actual, distributed system, and its 
thinking routine would run for a long time, it would probably be in the 
philosopher's own interest, as well as the system as a whole, that the 
philosopher would check for incoming fork requests as often as the thinking 
routine would allow. The reasoning is that the sooner a philosopher is given a 
fork, the sooner it will be done with it and thus ready to give it back.

If for instance two philosophers $A$ and $B$ would only handle fork requests at 
the very end of their thinking routines, then neither one of $A$ and $B$ could 
eat while the other was thinking; They both would have had to pass the forks 
while they were hungry and actually wanted the forks for themselves. Optimally 
two neighbors would pass the fork between them so that one was always eating 
while the other was thinking. In practice this would probably seldom be the 
case. Besides, the Dining Philosophers problem involves five processes, and no 
two neighbors can eat at the same time. Therefore at most two philosophers can 
eat simultaneously and there is at least one pair of neighbors where neither is 
eating.

What the philosophers are in fact doing during the eating and thinking routines 
is irrelevant to this model. But to make it appear as if *something* is 
happening, each state is realized by a corresponding method. The transition 
from one state to another takes place when the method corresponding to the 
first state calls the method corresponding to the second state.

In addition to `thinking` and `eating` there will be a state called `hungry` to 
clarify that philosopher is done thinking and will try to acquire the forks. 
The philosopher is then completely oblivious to the forks during the thinking 
state.

The calls to these state methods are done asynchronously[^so]. Each transition 
to a new state is therefore manifested by a new task in the COG before the task 
corresponding to the old state terminates. This automatically generates a 
scheduling point between each state.

Another benefit is that the transitions between states will become apparent in 
sequence diagrams, so that the diagrams give a more detailed picture of the 
model.

  [^so]: An infinite loop of recursive, synchronous method calls, would sooner 
  or later cause the Java backend to to crash with a *stack overflow*.

  [^suspendfail]: The most natural way to insert a scheduling point would 
  typically be to use the `suspend` statement, but this was opted out due to a 
  technical reason described in **ref**.



### The `thinking` and `eating` states {#sec:pstate}

An ABS task is unaware of the other tasks running on the same COG. Therefore, 
an incoming fork request cannot know the state of a philosopher without this 
being stored in a field on the `Philosopher` object. A simple algebraic data 
type `Pstate` is defined, enumerating the states `Thinking`, `Hungry` and 
`Eating`, and the philosophers are given a field of this type, called `state`. 
This field is updated to the next state at the end of the method corresponding 
to the *previous* state. 

The states *thinking* and *eating* are quite simple. Besides changing state, 
the method `think` does not do anything. After a philosopher is done eating, 
his forks should be marked as `dirty`, and the appropriate place to do this, is 
the `eat` method.

~~~{#lst:thinkeat float=t caption="The thinking and eating states of a hygienic philosopher"}
Unit think() {
    state = Hungry;
    this!hungry();
}

Unit eat() {
    forkL = Fork(forkID(forkL), Dirty);
    forkR = Fork(forkID(forkR), Dirty);
    state = Thinking;
    this!think();
}
~~~

### The `hungry` state {#sec:hungry}

The `hungry` state is where a philosopher will try to acquire the forks if he 
is not already holding them. Branches in the code are needed to cover each of 
the following possibilities, and in each case, requests must be sent to the 
appropriate neighbor(s): The philosopher may be missing *both* forks, only the 
*left* fork, or only the *right* fork. If none of these conditions are true, 
then the philosopher is already holding both the forks.

What is important to note, however, is that while the philosopher is waiting 
for one neighbor to pass a fork, the philosopher may himself receive a request 
from the opposite neighbor to pass the other fork. While it may seem 
inefficient, it is actually crucial for the algorithm that a philosopher will 
give up the fork he is holding while he is waiting for the other, unless, of 
course, the fork he is holding is clean.

If the philosophers refuse to give up their forks while they are waiting for 
their own fork request to resolve, a deadlock similar to the one in the 
original Dining Philosophers problem may occur. A sequence of events leading up 
to such a situation is illustrated with \Vref{fig:chandydl}.


\begin{figure}
    \hspace{-2cm}
    \begin{subfigure}[b]{\dimexpr\linewidth+2cm\relax}
        \parbox[b]{.5\linewidth}{%
            \includegraphics{figures/chandy-deadlock-sub1.eps}
        }%
        \hspace{.1\linewidth}%
        \parbox[b]{.4\linewidth}{%
            \caption{
                The initial configuration. The diagram is augmented with arrows 
                indicatig fork requests being sent after the philosophers are 
                getting hungry.
                An arrow pointing from a philosopher to a fork indicates a 
                request \emph{from} the philosopher \emph{for} that fork.}%
            \label{fig:chandyinit}%
        }
    \end{subfigure}
    \vspace{1cm}

    \hspace{-2cm}
    \begin{subfigure}[b]{\dimexpr\linewidth+2cm\relax}
        \parbox[b]{.5\linewidth}{%
            \includegraphics{figures/chandy-deadlock-sub2.eps}
        }%
        \hspace{.1\linewidth}%
        \parbox[b]{.4\linewidth}{%
            \caption{
                The configuration after the fork requests in the previous 
                subfigure have been resolved, and the philosophers are waiting 
                in a cycle, as described on the previous page.
                Clean forks are shown in solid black to indicate that they 
                currently cannot be taken from the philosopher holding them.}%
      %%%   XXX manual reference
            \label{fig:chandydlb}
            }%
    \end{subfigure}
    \vspace{1cm}
%
    \caption
    [Chandy/Misra \emph{hygienic} philosophers entering a cyclic configuration]
    {The initial configuration of Chandy and Misra's \emph{hygienic solution to 
    the diners problem} and how the philosophers can enter a cyclic 
    configuration, exhibiting the need for a philosopher to give up a (dirty) 
    fork while waiting for another.}
    \label{fig:chandydl}
\end{figure}


The topmost part of the figure shows the philosophers in the initial 
configuration. They are getting hungry one by one, starting to send fork 
requests to their neighbors. The first philosopher to get hungry is Philosopher 
1, the one that initially did not hold any forks. He sends the fork requests to 
his neighbors, philosophers 2 and 5. The first request reaches Philosopher 2, 
which sends over the fork just before getting hungry himself. Philosopher 2 
then sends a fork request back to Philosopher 1, and one request to Philosopher 
3, which in turn repeats this pattern. Philosopher 4 follows, successfully 
acquiring one of the two forks that Philosopher 5 started with.

After this, the configuration of forks and philosophers is as shown on the 
bottom part of the figure. Philosophers 1 through 4 are holding a clean fork in 
their right hand, waiting for the left fork. The only dirty fork, and thus the 
only fork that may change hands at this point, is the one held by Philosopher 
5. If Philosopher 5 gets hungry and sends a request for his left-hand fork 
before receiving the request from Philosopher 4, then all the philosophers are 
waiting in a cycle, similar to the deadlock situation explained in 
\Cref{deadlock}. From this it is clear that the philosophers need to accept 
incoming fork requests while they themselves are waiting for a fork.

<!-- XXX bad break of footnote -->
This is easily solved by using the await statement. Unlike in the 
implementation of the naive solution in \Cref{lst:naivep}, the remote procedure 
calls will be followed by await statements before using the `get` statement to 
obtain the return value[^awaitsyntax]. The process corresponding to the 
philosopher's `hungry` state is thereby suspended until the fork requests are 
resolved, allowing other tasks to be executed in the meanwhile.

  [^awaitsyntax]: When the philosopher only need to send one fork request, the 
  RPC immediately followed by `await` and `get` could normally be replaced by 
  the shorthand *await statement* which joins the three statements into one. 
  This is not done here because the Eclipse plug-in, which is used for making 
  the sequence diagrams, has not yet been updated to support this syntax.


Because there will be repeatedly need for checking whether a philosopher needs 
to send a fork request  a simple helper function `needsFork` is defined to 
shorten the code and make it more readable.

    def Bool needsFork(Fork f) = state(f) == RequestToken;

To ensure that a philosopher does not try to eat with only one fork, the forks 
must be checked again after the `hungry` method resumes from awaiting a fork 
request. An extra while loop wrapped around the main part of the `hungry` logic 
will suffice.


\begin{lstlisting} [
  caption=The \texttt{hungry} state of a hygienic philosopher,
  label=lst:hungry,
  style=numbered,
  %float=p
]
Unit hungry() {
    while (needsFork(forkL) || needsFork(forkR)) {

        if (needsFork(forkL) && needsFork(forkR)) {
            Fut<Fork> reqL = philL!request(forkID(forkL));
            Fut<Fork> reqR = philR!request(forkID(forkR));
            await reqL? & reqR?;
            forkL = reqL.get;
            forkR = reqR.get;

        } else if (needsFork(forkL)) {
            Fut<Fork> req = philL!request(forkID(forkL));
            await req?;
            forkL = req.get;

        } else if(needsFork(forkR)) {
            Fut<Fork> req = philR!request(forkID(forkR));
            await req?;
            forkR = req.get;
    } }
    state = Eating;
    this!eat();
}
\end{lstlisting}


### The fork request


With the logic of the philosophers' states *thinking*, *hungry* and *eating* in 
place, the only remaining piece is the handling of fork requests. Most of the 
foundation of this implementation has been laid, so the `request` method is 
quite straight-forward.

The method takes a `ForkID` argument, which the philosopher will need to check 
against the IDs of the two forks he is holding. In a more general solution the 
philosophers might have access to a larger, possibly variable, number of forks. 
It would then need a better facility for looking through the set of forks, such 
as a map structure. For this model, an if-statement for each of the forks will 
do. For simplicity it is assumed that a philosopher will not receive requests 
for other forks than the two that he has access to. <!-- XXX -->

The procedure of the fork request is the same for both forks, but different 
variables need to be accessed. Therefore the logic is duplicated in the two 
branches of the if-statement.

Having sorted out which of the two `Fork` fields the request is for, the method 
must wait for that fork to be `dirty` and for the philosopher to not be 
`eating`. These two conditions must be checked in one atomic operation; If the 
two conditions were instead checked in two individual steps, the process could 
pass the guards one by one, without both the conditions being true at the same 
time. The philosopher could then for instance end up giving away a fork right 
before it began eating.

To handle the request correctly and not break any invariants of the problem, a 
single `await` statement checks that the philosopher is not eating and that the 
fork is dirty. When the process has passed this guard, the field corresponding 
to the fork in question is changed to a request token, before a clean fork is 
returned to the caller of the fork request.

~~~{#forkrequest caption="The fork request"}
Fork request(ForkID i) {
    if (i == forkID(forkL)) {
        await this.state != Eating && state(forkL) == Dirty;
        forkL = Fork(i, RequestToken);

    } else if (i == forkID(fr)) {
        await this.state != Eating && state(forkR) == Dirty;
        forkR = Fork(i, RequestToken);
    }
    return Fork(i, Clean);
}
~~~


### Testing the model

The last adjustment to the model before initializing the philosophers, is to 
make sure that the philosophers enter the cycle of thinking, getting hungry, 
and eating. In the bottom of the `start` method in \Vref{lst:chandyp} an 
asynchronous call to `think()` must be added.

<!-- XXX "main method?" -->
In the main block the five philosophers are created, named `p1` through `p5`. 
To match the initial configuration as it was shown in \Cref{fig:chandyinit},
each philosopher has the fork with the number corresponding to his name, on his 
left-hand side. And for each pair of neighbors, the one with the highest number 
will be the one to hold the fork they share between them. Thus `p5` is the one 
initialized with two forks, and `p1` is initialized without any forks.

After all philosophers have been initialized, their `start` routines can be 
called. The order of the parameters to this method will determine which 
philosophers that are neighbors. To match with the mapping of the philosophers' 
names and the forks' IDs as defined above, the neighbor relationship is set up 
as follows.

    // Abbbritated to make each statement fit on one line
    Phil p1 = new Phil(Fork(1, Token), Fork(2, Token));
    Phil p2 = new Phil(Fork(2, Token), Fork(3, Token));
    Phil p3 = new Phil(Fork(3, Token), Fork(4, Token));
    Phil p4 = new Phil(Fork(4, Token), Fork(5, Token));
    Phil p5 = new Phil(Fork(5, Token), Fork(1, Token));

    p1!start(p5, p2);
    p2!start(p1, p3);
    p3!start(p2, p4);
    p4!start(p3, p5);
    p5!start(p4, p1);



#### Deadlock

It was explained in \Cref{sec:hungry} how the system could enter a 
configuration where all the philosophers were waiting in a cyclic arrangement. 
A deadlock similar to the one in the naive solution of the Dining Philosophers 
would occur if the philosophers would not be able to handle fork requests while 
they were in the *hungry* state waiting for another fork.

With the interactive scheduler in the Eclipse plug-in, such a situation could 
be reproduced. \Vref{fig:hungrygrab} shows that the `hungry` method works as 
intended, allowing concurrent fork requests to be handled. For clarity the two 
other philosopher not touched by the two fork requests were edited out of the 
figure.

The figure also shows the different states of the two leftmost philosophers, 
thanks to the states being their own processes, and it can be seen that the 
final fork request is not granted until after the philosopher is done eating.

It has been shown that a dirty fork can always change owner. Therefore any 
configuration with at least one dirty fork can always change into another 
configuration; There cannot be a deadlock in the system as long as there are 
dirty forks.

\Cref{fig:chandydlb} shows a configuration where all forks but one are clean. 
This configuration is reached from the initial configuration by having the 
philosophers 2 through 5 give up their left-hand forks. In order to reach a 
configuration where all forks are clean, Philosopher 5 must also give up his 
right-hand fork. No forks can then immediately change hands, but Philosopher 1 
is now holding two forks, and may eat, so the configuration still advances. 
After Philosopher 1 has eaten, his forks are again dirty. Because Philosopher 5 
is initially holding two forks, a configuration where each philosopher is 
holding one clean fork cannot be reached. Deadlock is therefore impossible.




<!-- TODO noe om at deadlock dermed ikke kan oppstå; Så lenge en gaffel er 
skitten, kan den skifte eier. Det går ikke at alle filosofene har en gaffel 
hver og alle gafflene er rene. Det nærmeste r situasjonen som ble nevnt, og den 
ordner seg altså.

For at alle gaflene skal være rene på en gang, må alle skifte eier. Siden en 
filosof har to gaffler, vil og én filosof ende opp med to rene gaffler, og han 
kan spise.

dette er den eneste måten å entre en situasjon hvor alle filosofene har én 
gaffel hver;

-->


\begin{figure}
    \stretchgraphics[center]{2cm}{figures/hungrygrab-crop.pdf}
    \caption
    [Hygienic philosopher giving up one fork while waiting for another]
    {A simplified sequence diagram showing a hungry Chandy/Misra philosopher 
    (in the middle) waiting to recive a fork from his right-hand neighbor. In 
    the meantime the opposite neighbor sends a request for the other fork. In 
    order to prevent a possible deadlock the philosopher gives that fork away, 
    and must wait until the left-hand neighbor is done eating}
    \label{fig:hungrygrab}
\end{figure}


<!--
#### put this somewhere before `hungry`?

By naming the arguments to the `Fork` data type, and having only one 
constructor for it, functions for fetching the ID and the state of the forks 
are automatically generated, and completely safe to use. There is one task that 
will be performed on multiple occasions, namely checking that a philosopher is 
currently holding a given fork. For readability and code reuse a function 
`hasFork` is defined.

    def Bool hasFork(Fork f) = state(f) != RequestToken;

#### place this after the hungry state has been covered:
Setting the `state` field to the *next* state in each of these methods makes 
sure that for instance a philosopher is concidered eating for as long as there 
is an `eat()` task on the COG; If there 

Of course no other task can be activated until the current method has 
terminated.

For each state transition there will appear a little overlap before the current 
state's method terminates, but there is no new task 

will not give up his forks to another philosopher as long as there is a `eat()` 
task on the COG.
-->



#### Fairness

\Cref{starvation} covered the principle of process starvation, and it was shown 
how the scheduler could choose

XXX Skip this?

This is a stricter requirements

TODO

* Scheduler seems to be weakly fair
    * Show that running the model on the Java backend tends to end in a 
      situation similar to \Cref{fig:chandydl}, having one philosopher thinking 
      and eating non-stop, the rest locked.
    * Show how to solve that with an additional "fairness guard"
    * Ref to \Cref{starvation} -- without additional fairness, this model is 
      actually a better example of starvation!
    * Ref [@andrews2000, p.74]



<!--
# draft/some ideas for the rest of the chapter


