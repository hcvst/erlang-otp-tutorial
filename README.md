Originaly published on December 2012 at http://blog.bot.co.za/en/article/349/an-erlang-otp-tutorial-for-beginners

# An Erlang OTP tutorial for beginners — A bot blog

This is an Erlang/OTP tutorial for novice Erlang programmers. If you are familiar with Erlang's syntax, know how to compile modules and are curious to discover OTP and how to leverage its power in your own Erlang applications this tutorial is for you. To kick off we are going to create a non-OTP server and supervisor, examine their shortcomings and then turn our code into a proper OTP application. 

This tutorial will look at:

* How to build a non-OTP server from scratch and how to automatically restart it using a non-OTP supervisor.
* What the shortcomings of our implementation are and how to addresses them with OTP.
* Very briefly, tools such as _rebar_ to smothen our OTP journey and OTP's folder conventions.
* How to use OTP's _gen_server_ and _supervisor_ behaviour.
* The OTP _application_ behaviour to tie it all together.

First, however, some introductions are in order.

## About you

I assume that you are familiar with Erlang's syntax, that you know what Erlang modules are and how to compile them and that you understand functional programming concepts such as recursion and single-assignment variables. You do? Great! Let's learn some OTP. It is not difficult to get started, as you will find out shortly, but first let me tell you briefly about my own Erlang experience.

## About me

I am pretty much on the same page as you - except perhaps, that I've started to read Manning's Erlang & OTP in action. I am not exaggerating when I say that OTP in action is one of my favourite technical books. It contains a wealth of easy to absorb information. [Buy a copy][1] if you can.

'Scaffolding', I think, is what they called it in psychology class when peers teach peers. I am writing this as much to cement my own undertstanding of OTP as to help kickstart yours. Let's study together. Please leave a comment whenever you think I am wrong.

## About OTP

OTP stands for Open Telecom Platform but in spite of its name it is not tied to telecom applications whatsoever. Think of OTP as framework to help you write robust and highly available applications on the Erlang platform.

Specifially, you might like to think of OTP in terms of the following concepts:

* a specification (called a 'behaviour') that dictates which functions your module should export
* a concrete module that declares that it conforms to a behaviour and implements these functions
* a runtime component that can execute behaviour-compliant modules within the OTP context

If you are coming from on Object Oriented world such as Jave these concepts roughly translate to:

* an interface or abstract class that defines a behaviour
* a concrete derived implementation of that interface or abstract class
* a container that knows how to instrument instances of such concrete implementations

Please understand that these are, of course, only superficial similarities. They might help us understand new concepts in terms of old knowledge, but, make no mistake, we are comparing apples and oranges.

## A non OTP server

With the introductions out of the way let's make sure that we really are on the same page by writing a simple server that does not use OTP. Here's the code:
    
    
    %%%---------------------------------------------------------------------------
    %%% @doc A simple server that does not use OTP. 
    %%% @author Hans Christian v. Stockhausen 
    %%% @end
    %%%---------------------------------------------------------------------------
    
    -module(no_otp).                     % module name (same as our erl filename, 
                                         % i.e. no_otp.erl) 
    
    %% API
    -export([                            % the functions we export - our API 
      start/0,                           % - start new server process
      stop/0,                            % - stop server 
      say_hello/0,                       % - print "Hello" to stdout 
      get_count/0                        % - reply with number of times the main  
      ]).                                %   loop was executed
    
    %% Callbacks
    -export([init/0]).                   % exported so we can spawn it in start/0
                                         % listed as a separate export here as a
                                         % hint to clients not to use it
    
    -define(SERVER, ?MODULE).            % SERVER macro is an alias for MODULE, 
                                         % and expands to 'no_otp'
    
    -record(state, {count}).             % record for our server state. Rather
                                         % arbitrarily we track how often the
                                         % main loop is run - see loop/1
    
    
    %============================================================================= 
    % API - functions that our clients use to interact with the server 
    %=============================================================================
    
    start() ->                           % spawn new process that calls init/0   
      spawn(?MODULE, init, []).          % the new process' PID is returned
    
    stop() ->                            % send the atom stop to the process                            
      ?SERVER ! stop,                    % to instruct our server to shut down
      ok.                                % and return ok to caller
    
    say_hello() ->                       % send the atom say_hello to the process
      ?SERVER ! say_hello,               % to print "Hello" to sdout
      ok.
    
    get_count() ->                       % send callers Pid and the atom get_count
      ?SERVER ! {self(), get_count},     % to request counter value
      receive                            % wait for matching response and return
        {count, Value} -> Value          % Value to the caller 
      end.  
    
    
    %=============================================================================
    % callback functions - not to be used by clients directly
    %============================================================================= 
    
    init() ->                            % invoked by start/0 
      register(?SERVER, self()),         % register new process PID under no_otp 
      loop(#state{count=0}).             % start main server loop 
    
    %=============================================================================
    % internal functions - note, these functions are not exported 
    %============================================================================= 
    
    loop(#state{count=Count}) ->         % the main server loop
      receive Msg ->                     % when API functions send a message 
        case Msg of                      % check the atom contained 
          stop ->                        % if atom is 'stop'
            exit(normal);                %   exit the process 
          say_hello ->                   % if atom is 'say_hello'
            io:format("Hello~n");        %   write "Hello" to stdout
          {From, get_count} ->           % if Msg is tuple {Pid(), get_count}
            From ! {count, Count}        % reply with current counter value
        end                              %   in tagged tupple {count, Value}
      end,
      loop(#state{count = Count + 1}).   % recurse with updated state

## Let's try our server

* Save the code above to a file named _no_otp.erl_.
* Start the Erlang shell with erl in the same directory.
* Compile the module: c(no_otp).
* Start the server: no_otp:start().
* Instruct the server to print "Hello" a couple of times: no_otp:say_hello().
* Ask for the current invocation counter value: no_otp:get_count().
* Shut the server down: no_otp:stop().

Here is my session. Admittedly, one has to wonder what's the point of get_count/0. There is none, except to demonstrate how to implement a synchronous function.
    
    
    Eshell V5.8.5  (abort with ^G)
    1> c(no_otp).
    {ok,no_otp}
    2> no_otp:start().
    <0.39.0>
    3> no_otp:get_count().
    0
    4> no_otp:say_hello().
    Hello
    ok
    5> no_otp:get_count().
    2
    6>
    

I initially found the term _server_ in this context a little confusing. How is this a server? On second thought, it is not at all unlike the HTTP server that served this page to your browser. It runs continously until you stop it, it speaks a protocol (not HTTP but Erlang's internal messaging protocol) and responds to inputs with side-effects or response messages. It's a server, alright.

I am not going to discuss the code here in detail, since the comments already do that, I hope. However, I would like to briefly outline the structure:

* Implementation details are hidden from clients by a thin API layer. The functions start/0, stop/0, etc.. decouple clients from the internal protocol (i.e. the atoms 'start', 'stop', … that these functions send) as well as from internal functions. As a result we could modify either or both without breaking client code.
* There exists a function init/0 that sets up the initial state and starts the main server loop. It also registeres our server, by convention, under the module name and could perform additional initialisation steps such as opening a database connection. Note that this function is exported by our module even though it is not intended to be called by client code. Otherwise our call to spawn/3 in start/0 does not work.
* There is a main loop that waits for messages, performs some actions in response, optionally updates the server state and finally calls itself.

## What's wrong with this server?

Not much really. Thanks to Erlang it should run and run and run once started. This is a such a simple server, so we can be quite confident that our implementation is correct. Also, thanks to our API, we can be certain that clients don't inadvertently crash it by sending malformed data - or can we?

Start up the server as before and then type no_otp ! crash. at the shell.

Ouch! Since the server is registered under the name no_otp (see init/0) we can send arbitrary messages to it and because our code does not have a matching case clause for 'crash': it crashes.

What to do? We could write a case clause to match any message and simply ignore it. However, that would be like a catch-all-clause when handling Exceptions, which we know is evil and likely to obscure bugs.

Erlang follows a philosophy of 'let it crash'. There we go again. I am sure you've read that phrase more than once. But let's recap what that means for you, the Erlang programmer: Don't program defensively, don't try to anticipate all circumstances, focus on your specification, on what the code should do and keep it compact and succinct. If there are bugs in your code, it will crash and you can fix it. If it crashes, because of bugs in your client's code, let it crash and restart in a known good state, rather than trying to keep it alive at all costs. Don't complicate the code by trying to sort out your client's mess. It is theirs to fix. Got it?

## Supervision

Fine, I'll let it crash, but how can I automatically restart the server when that happens? The answer is to code another process, that monitors our server and restarts it when it detects that it died. Such a process is a _supervisor_ and what follows is the code for our non-OTP version.

## A non-OTP supervisor

Before I show you the supervisor code have a look at the annotated Erlang shell session below to see how to exercise it. The function whereis/0 returns the Pid of the process that is registered under a particular name. We can see that sup:start_server/0 must in fact have started our _no_otp_ server, since it shows up under process ID <0.40.0>. Automatic restart also seems to work as it should.
    
    
    Eshell V5.8.5  (abort with ^G)
    1> c(sup).                                   % compile our supervisor module
    {ok,sup}
    2> sup:start_server().                       % start the supervisor
    <0.39.0>                                     % it got the process ID ..39..
    3> whereis(no_otp).                          % let's find our no_otp process
    <0.40.0>                                     % it got ..40..
    4> no_otp:say_hello().                       % let's try to use the API
    Hello                                        % works
    ok
    5> no_otp:get_count().                       % works too 
    1
    6> no_otp ! crash.                           % time to crash it
    crash
    
    =ERROR REPORT===
    Error in process <0.40.0> with exit value: 
      {{case_clause,crash},[{no_otp,loop,1}]}   % as expected a case_clause error
    
    7> no_otp:get_count().                      % but a new process was started
    0                                           % as we can see here
    8> no_otp:stop().                           % now let's stop it 'normally'
    ok
    9> whereis(no_otp).                         % and check for a new instance
    undefined                                   % but there is none. Correct!
    10>
    

Now, for the supervisor code:
    
    
    %%%----------------------------------------------------------------------------
    %%% @doc A non-OTP supervisor
    %%% @author Hans Christian v. Stockhausen 
    %%% @end
    %%%----------------------------------------------------------------------------
    
    -module(sup).
    
    %% API
    -export([start_server/0]).
    
    %% Callbacks
    -export([supervise/0]).
    
    %%=============================================================================
    %% API
    %%=============================================================================
    
    start_server() ->
      spawn(?MODULE, supervise, []).
    
    %%=============================================================================
    %% Callbacks
    %%=============================================================================
    
    supervise() ->
      process_flag(trap_exit, true),    % don't crash us but rather send us a
                                        %  message when a linked process dies
      Pid = no_otp:start(),             % start our server
      link(Pid),                        % and link to it
      receive {'EXIT', Pid, Reason} ->  % wait for a message that informs us
        case Reason of                  %  that our porcess shut down
          normal -> ok;                 % if the reason is 'normal' do nothing
          _Other -> start_server()      %  otherwise start all over 
       end
      end.
    

As I wrote in the introduction, I am an Erlang novice myself. I have little confidence in the supervisor code above, and **that's the point**. If we want to write applications the 'Erlang way', if we want to embrace the philosophy of 'let it crash', we have to structure our code in terms of servers, supervisors and supervisors' supervisors. That's a lot of structural code and a lot of repetitive boilerplate with amble opportunity to introduce bugs. The concept of servers and supervision hierarchies is easy enough to understand. However, is it also easy to get 100% right?

## OTP to the rescue

Fortunately for us, someone else has done the hard work already. There is OTP, a framework specifically written to help us compose servers, supervisors and other components, which we will examine at a later stage. The OTP developers have given us a rock solid framework to hook our own, possibly brittle code, into a framework that is tested, battle-hardend and known to work.

It is time to rewrite our server and supervisor as OTP behaviours. But firstly, let's look at two tools, to help us along the way.

## Rebar & Emacs

Just because OTP saves us from having to reinvent supervision hierarchies there is, as you will see, still a fair amount of code that one has to type. A lot remains boilerplate and gets real tideous real fast. Also, you want to get into the good habit of documenting your code with _Edoc annotations_, which results in even more repetitive typing. Hence, a little tool support is in order.

**Rebar** is a build tool for Erlang. Check their [github wiki page][2] for detailed instructions. I am only going to cover what we need now.

Run the following commands from your nix Shell to install rebar and to create the scaffolding for our app 'hello'.
    
    
    $> mkdir hello
    $> cd hello    
    $> wget https://raw.github.com/wiki/rebar/rebar/rebar
    $> chmod u+x rebar
    $> ./rebar create-app appid=hello 
    

If you list the contents of the newly created _src_ directory you will see that the following files have been generated:

* _hello_app.erl_ \- OTP application behaviour (ignore for now)
* _hello.app.src_ \- OTP application configuration template (also ignore please)
* _hello_sup.erl_ \- minimal root supervisor implementation

What's missing is a file to contain our server code. Let's ask _rebar_ to generate one for us.
    
    
    $> ./rebar create template=simplesrv srvid=hello_server
    

If you don't want to use rebar you can, of course, create the files manually and place them into your _src_ folder. It's convention to call the folder _src_, as I will explain shortly.

**Emacs** offers an Erlang mode that really speeds up your Edit-Compile-Run cycle and also comes with templates. I mostly use _vi_, for day-to-day editing tasks, but was always looking for a reason to try _emacs_. I've found one in its Erlang mode and suggest you try it too, if you haven't done so already. Try it now:
* emacs src/hello_sup.erl
* hit **F10** to open the menu
* type **e** for Erlang
* type **c** to select the **compile** option
* type **c** again to compile _hello_sup.erl_ (Note the shortcut C-c C-k for future reference)
* Inspect the compiler output
* press **C-0** to close the compiler window
* press **C-x C-c** to exit _emacs_

Powerfull stuff and we've barely scratched the surface, I am sure, but feel free to follow along with your favourite editor.

Before we finally implement our OTP behaviours, we have to look at OTP folder layout.

## OTP folders

To create the OTP folder structure use the following command from bash, alternatively **rebar** can create them for you. Only _src_ and _ebin_ are required.
    
    
    mkdir -p my_app/{doc,ebin,priv,include,src,test}
    

* _doc_ \- for documentation which typically is generated automatically
* _ebin_ \- your compiled code (.beam)
* _priv_ \- resource files such as html, images etc.
* _include_ \- public header files that client code may use (.hrl)
* _src_ \- source code and private header files (.erl, .hrl)
* _test_ \- tests (_test.erl)

## OTP gen_server

As the name implies, the _gen_server_ behaviour helps us write generic servers. Here's our server rewritten as OTP behaviour.
    
    
    %%%----------------------------------------------------------------------------
    %%% @doc An OTP gen_server example
    %%% @author Hans Christian v. Stockhausen 
    %%% @end
    %%%----------------------------------------------------------------------------
    
    -module(hello_server).         % Nothing new in this section except for the
                                   %  next line where we tell the compiler that
    -behaviour(gen_server).        %  this module implements the gen_server
                                   %  behaviour. The compiler will warn us if
    -define(SERVER, ?MODULE).      %  we do not provide all callback functions
                                   %  the behaviour announces. It knows what
    -record(state, {count}).       %  functions to expect by calling 
                                   %  gen_server:behaviour_info(callbacks). Try it.
    %%-----------------------------------------------------------------------------
    %% API Function Exports
    %%-----------------------------------------------------------------------------
    
    -export([                      % Here we define our API functions as before 
      start_link/0,                % - starts and links the process in one step
      stop/0,                      % - stops it
      say_hello/0,                 % - prints "Hello" to stdout
      get_count/0]).               % - returns the count state
    
    %% ---------------------------------------------------------------------------
    %% gen_server Function Exports
    %% ---------------------------------------------------------------------------
    
    -export([                      % The behaviour callbacks
      init/1,                      % - initializes our process
      handle_call/3,               % - handles synchronous calls (with response)
      handle_cast/2,               % - handles asynchronous calls  (no response)
      handle_info/2,               % - handles out of band messages (sent with !)
      terminate/2,                 % - is called on shut-down
      code_change/3]).             % - called to handle code changes
    
    %% ---------------------------------------------------------------------------
    %% API Function Definitions
    %% ---------------------------------------------------------------------------
    
    start_link() ->                % start_link spawns and links to a new 
        gen_server:start_link(     %  process in one atomic step. The parameters:
          {local, ?SERVER},        %  - name to register the process under locally
          ?MODULE,                 %  - the module to find the init/1 callback in 
          [],                      %  - what parameters to pass to init/1
          []).                     %  - additional options to start_link
    
    stop() ->                      % Note that we do not use ! anymore. Instead
        gen_server:cast(           %  we use cast to send a message asynch. to
          ?SERVER,                 %  the registered name. It is asynchronous
          stop).                   %  because we do not expect a response.
    
    say_hello() ->                 % Pretty much the same as stop above except
        gen_server:cast(           %  that we send the atom say_hello instead.
          ?SERVER,                 %  Again we do not expect a response but
          say_hello).              %  are only interested in the side effect.
    
    get_count() ->                 % Here, on the other hand, we do expect a 
        gen_server:call(           %  response, which is why we use call to
          ?SERVER,                 %  synchronously invoke our server. The call 
          get_count).              %  blocks until we get the response. Note how
                                   %  gen_server:call/2 hides the send/receive
                                   %  logic from us. Nice.
    %% ---------------------------------------------------------------------------
    %% gen_server Function Definitions
    %% ---------------------------------------------------------------------------
    
    init([]) ->                    % these are the behaviour callbacks. init/1 is
        {ok, #state{count=0}}.     % called in response to gen_server:start_link/4
                                   % and we are expected to initialize state.
    
    handle_call(get_count, _From, #state{count=Count}) -> 
        {reply, 
         Count,                    % here we synchronously respond with Count
         #state{count=Count+1}     % and also update state
        }.
    
    handle_cast(stop, State) ->    % this is the first handle_case clause that
        {stop,                     % deals with the stop atom. We instruct the
         normal,                   % gen_server to stop normally and return
         State                     % the current State unchanged.
        };                         % Note: the semicolon here....
    
    handle_cast(say_hello, State) -> % ... becuase the second clause is here to
        io:format("Hello~n"),      % handle the say_hello atom by printing "Hello"
        {noreply,                  % again, this is asynch, so noreply and we also
        #state{count=
          State#state.count+1}
        }.                         % update our state here
    
    handle_info(Info, State) ->      % handle_info deals with out-of-band msgs, ie
        error_logger:info_msg("~p~n", [Info]), % msgs that weren't sent via cast
        {noreply, State}.          % or call. Here we simply log such messages.
    
    terminate(_Reason, _State) ->  % terminate is invoked by the gen_server
        error_logger:info_msg("terminating~n"), % container on shutdown.
        ok.                        % we log it and acknowledge with ok.
    
    code_change(_OldVsn, State, _Extra) -> % called during release up/down-
        {ok, State}.               % grade to update internal state. 
    
    %% ------------------------------------------------------------------
    %% Internal Function Definitions
    %% ------------------------------------------------------------------
    
    % we don't have any.
    

To compile our OTP server, change into the application directory (i.e. the one above _src_) and run ./rebar compile or, if you decided not to use _rebar_, run erlc -o ebin src/*.erl instead. _rebar_ automatically creates the _ebin_ directory, so if you use _erlc_, you need to create it manually first.

Let's take our OTP server for a test ride. Here's my output:
    
    
    $> erl -pa ebin                # start erl and -pa (path add) our ebin     
    1> hello_server:start_link().  % start our server. 
    {ok,<0.34.0>}
    2> hello_server:say_hello().   % try the say_hello/0 API function
    Hello
    ok
    3> hello_server:get_count().   % observe how the server keeps track 
    1
    4> hello_server:get_count().   % of state
    2
    5> hello_server ! stop.        % let's send an out-of-band message
    
    =INFO REPORT===                % here's the output from error_logger
    stop
    stop
    6> hello_server:stop().        % now let's call our stop/0 API function
    
    =INFO REPORT===                % this is the output from terminate
    terminating
    ok
    7> whereis(hello_server).      % and as we can see the process no longer
    undefined                      % exists.
    8>
    

So, what have we gained by turning our non-OTP version into an OTP _gen_server_?

* First of all, anyone familiar with _gen_server_ can quickly understand how our code is organized, what API we expose and where to look for implementation details. This is the power of patterns.
* Three distinct messaging mechanisms: synchronous, asynchronous and out-of-band.

But there's more. If, as the following session shows, we start the SASL application (System Application Support Libraries) and then force a crash, we get a valuable error report, something our non-OTP version does not benefit from.
    
    
    1> application:start(sasl).                    % let's start SASL
    ok
    2>                                         
    =PROGRESS REPORT====                           % I've deleted all startup messages
              application: sasl                    % except for the last one here 
              started_at: nonode@nohost            % for brevity. SASL is up.
    
    2> hello_server:start_link().                  % let's start our server
    {ok,<0.44.0>}
    3> whereis(hello_server).                      % and check that it was registered
    <0.44.0>
    4> gen_server:cast(hello_server, crash).       % now let's crash it
    
    =INFO REPORT==== 17-Dec-2012::11:00:56 ===     % excellent, gen_server called terminate 
    terminating                                    % so we could clean up, close sockets, etc.
    
    =ERROR REPORT==== 17-Dec-2012::11:00:56 ===    % what follows is a SASL report that we
    ** Generic server hello_server terminating     % for free, simply by using OTP.
    ** Last message in was {'$gen_cast',crash}
    ** When Server state == {state,0}
    ** Reason for termination ==                   % v--- aha - here's what went wrong 
    ** {function_clause,[{hello_server,handle_cast,[crash,{state,0}]},
                         {gen_server,handle_msg,5},
                         {proc_lib,init_p_do_apply,3}]}
    
    =CRASH REPORT==== 17-Dec-2012::11:00:56 ===    % and here's even more info on the environent
      crasher:
        initial call: hello_server:init/1
        pid: <0.44.0>
        registered_name: hello_server
        exception exit: {function_clause,
                            [{hello_server,handle_cast,[crash,{state,0}]},
                             {gen_server,handle_msg,5},
                             {proc_lib,init_p_do_apply,3}]}
          in function  gen_server:terminate/6
        ancestors: [<0.32.0>]
        messages: []
        links: [<0.32.0>]
        dictionary: []
        trap_exit: false
        status: running
        heap_size: 377
        stack_size: 24
        reductions: 142
      neighbours:
        neighbour: [{pid,<0.32.0>},
                      {registered_name,[]},
                      {initial_call,{erlang,apply,2}},
                      {current_function,{io,wait_io_mon_reply,2}},
                      {ancestors,[]},
                      {messages,[]},
                      {links,[<0.26.0>,<0.44.0>]},
                      {dictionary,[]},
                      {trap_exit,false},
                      {status,waiting},
                      {heap_size,2584},
                      {stack_size,30},
                      {reductions,6595}]
    ** exception exit: function_clause
         in function  hello_server:handle_cast/2
            called as hello_server:handle_cast(crash,{state,0})
         in call from gen_server:handle_msg/5
         in call from proc_lib:init_p_do_apply/3
    5> whereis(hello_server).                      % our process crashed, but at least the
    undefined                                      % report above gives us an idea why.
    

## OTP supervisor

Here's the code that _rebar_ generated for us. It is a supervisor template and not yet customized.
    
    
    -module(hello_sup).
    
    -behaviour(supervisor).
    
    %% API
    -export([start_link/0]).
    
    %% Supervisor callbacks
    -export([init/1]).
    
    %% Helper macro for declaring children of supervisor
    -define(CHILD(I, Type), {I, {I, start_link, []}, permanent, 5000, Type, [I]}).
    
    %% ===================================================================
    %% API functions
    %% ===================================================================
    
    start_link() ->
        supervisor:start_link({local, ?MODULE}, ?MODULE, []).
    
    %% ===================================================================
    %% Supervisor callbacks
    %% ===================================================================
    
    init([]) ->
        {ok, { {one_for_one, 5, 10}, []} }.
    

Wow, that's short! Wow, what on earth does it mean? We're going to dissect it in a moment and then fill in the gap to make this template work with our _gen_server_, but even if we don't understand the details yet there's a lot we understand already.

* This module implements the OTP _supervisor_ behaviour
* It has one API function start_link/0 and one behaviour callback function init/1.
* There is only one single function call in this entire listing (supervisor:start_link/3)
* init/1, where we expect the magic to happen, only returns a tuple.

Well, init/1 **is** where the magic happens. But in contrast to our non-OTP supervisor, where we had to code the logic to monitor and restart our sever, we are using a purely declarative, albeit still cryptic approach, in form of the tuple, to configure the supervisor container.

What's going on with that -define(CHILD(I, Type), ...)? It is a macro, just like the familar -define(SERVER, ?MODULE). macro, except that this one takes two arguments. You use this macro by prefixing it with ? and by providing values for _I_ and _Type_. So, for example, ?CHILD(hello_server, worker) would be expanded to {hello_server, {hello_server, start_link, []}, permanent, 5000, worker, [hello_server]} at compile time. But the macro isn't used anywhere in our code yet. Where are we supposed to use it?

Let's take a look at the _supervisor_ template _emacs_ generates for us. We are only interested in the init/1 function, which is why I omitted the rest. But try to use _emacs_ to generate a supervisor template yourself. To discover what goodness I'm hiding hit: F10, e, S, 0, where e stands for Erlang, S for Skeletons and 0 selects the supervisor template. Here's what I got:
    
    
    init([]) ->
        RestartStrategy = one_for_one,
        MaxRestarts = 1000,
        MaxSecondsBetweenRestarts = 3600,
    
        SupFlags = {RestartStrategy, MaxRestarts, MaxSecondsBetweenRestarts},
    
        Restart = permanent,
        Shutdown = 2000,
        Type = worker,
    
        AChild = {'AName', {'AModule', start_link, []},
              Restart, Shutdown, Type, ['AModule']},
    
        {ok, {SupFlags, [AChild]}}.
    

This is probably easier to understand, than what _rebar_ generated for us. Although it should, of course, at least stucturally, boil down to the same configuration tuple. Let's substitute both atoms 'AName' and 'AModule' with hello_server and resolve the last tuple fully. I am going to insert some line breaks and comments to make it easier to see what goes where.
    
    
    {ok,                      % ok, supervisor here's what we want you to do
      {                       
        {                     % Global supervisor options
          one_for_one,        % - use the one-for-one restart strategy
          1000,               % - and allow a maximum of 1000 restarts
          3600                % - per hour for each child process
        },                     
        [                     % The list of child processes you should supervise
          {                   % We only have one
            hello_server,     % - Register it under the name hello_server
            {                 % - Here's how to find and start this child's code 
              hello_server,   %   * the module is called hello_server
              start_link,     %   * the function to invoke is called start_link
              []              %   * and here's the list of default parameters to use
            },                
            permanent,        % - child should run permantenly, restart on crash 
            2000,             % - give child 2 sec to clean up on system stop, then kill 
            worker            % - FYI, this child is a worker, not a supervisor
            [hello_server]    % - these are the modules the process uses  
          } 
        ]                     
      }                        
    }.                        
    

I leave it up to you to decide on the format you prefer. You can use _rebar_'s compact notation, _emac_'s verbose template or even the resolved snippet above in your supervisor's init/1. As an exercise, please implement your supervisor now and compile it. However, before we test it, let's look at the parameters we used in the snippet above.

* **one_for_one**: This restart strategy instructs the supervisor to restart the crashed child process only. In our case there is only one, but if there were more supervised child processes, they would not be affected. In contrast, a restart strategy of _one_for_all_ automatically restarts all children if one crashes. For autonomous child processes use _one_for_one_. If, on the other hand, child processes form a tightly coupled subsystem it may be better to restart the entire subsystem with _one_for_all_. Consult man 3erl supervisor or this [page][3] for additional information. In particular, look at the _simple_one_for_one_ strategy which allows you to use a supervisor as a process factory.
* **permanent**: This tells the supervisor that the child should always be restarted (subject to the global restart rules). Other Restart values are _temporary_ and _transient_ . _temporary_ processes are never restarted but _transient_ child processes are if they terminate abnormally (i.e. they crashed). For our use-case _permanent_ makes sense, but in the case of a process factory, _transient_ or _temporary_ may be more applicable.
* **worker**: Here we want to supervise a _gen_server_ \- a worker - but it is common to build supervisor hierarchies. Setting this value to _supervisor_ instead informs the supervisor that the child is also a supervisor. This is to help it manage child processes optimally.

Let's try our OTP supervisor.
    
    
    1> hello_sup:start_link().                % let's start our root supervisor
    {ok,<0.34.0>}
    2> whereis(hello_server).                 % and ensure that it started out child process
    <0.35.0>
    3> hello_server:say_hello().              % now let's call the child API
    Hello
    ok
    4> hello_server:get_count().              % and verify that all works as before
    1
    5> gen_server:cast(hello_server, crash).  % time to crash the worker process
    
    =INFO REPORT====                          % good, terminate is still being called
    terminating
    
    =ERROR REPORT==== 17-Dec-2012::13:55:51 ===
    ** Generic server hello_server terminating
    ** Last message in was {'$gen_cast',crash}
    ** When Server state == {state,2}
    ** Reason for termination ==             % and --v-- here we see why it crashed
    ** {function_clause,[{hello_server,handle_cast,[crash,{state,2}]},
                         {gen_server,handle_msg,5},
                         {proc_lib,init_p_do_apply,3}]}
    ok
    6> whereis(hello_server).                % but look, a new process was already started
    <0.40.0>
    7> hello_server:get_count().             % obviously, with a new state
    0
    8> hello_server:stop().                  % let's call our API stop function
    
    =INFO REPORT===
    terminating                              % looking good               
    ok
    9> whereis(hello_server).                % but our supervisor started another server.
    <0.44.0>
    10>
    

It works as designed, although, perhaps, not as expected. The call to stop_server/0 stops our process, but the supervisor immediately starts a new instance. We could change the Restart parameter in our supervisor's Child specification to _transient_, but then we are left with a childless supervisor. I don't think this is a problem though, but rather the result of our somewhat arbitrary "Hello" scenario. Let's turn our attention therefore to **OTP applications** to learn how to start and stop applications as a whole.

## OTP application

We have already seen an example of an application in this post: SASL, which we launched with application:start(sasl). Now it is time to find out how we can bundle our root supervisor and our server in an application so that we can start it in the same manner.

Unlike our _gen_server_ and _supervisor_ implementation, the OTP _application_ behaviour requires two files. A configuration file, conventially named, _appname.app_ and an Erlang source file called _appname_app.erl_. In our case these are:

* _ebin/hello.app_
* _src/hello_app.erl_

Note that the _hello.app_ file is placed into the _ebin_ directory. This makes sense. The _.app_ file tells Erlang how to launch our application. Since we might only ship our app's compiled _.beam_ files, not its source, this is where the _.app_ file naturally belongs.

As a _rebar_ users you will notice that there exists also a _src/hello.app.src_ file, as mentioned earlier. This file serves as a template for your _ebin/hello.app_ file when you run ./rebar compile.

Here is my _.erl_ and _.app_ file as generated by _rebar_.
    
    
    -module(hello_app).
    
    -behaviour(application).
    
    %% Application callbacks
    -export([start/2, stop/1]).
    
    %% ===================================================================
    %% Application callbacks
    %% ===================================================================
    
    start(_StartType, _StartArgs) ->
        hello_sup:start_link().
    
    stop(_State) ->
        ok.
    

This is the _hello_app.erl_ file that implements the OTP _application_ behaviour. Need I say more?
    
    
    {application,hello,
      [{description,[]},
       {vsn,"1"},
       {registered,[]},
       {applications,[kernel,stdlib]},
       {mod,{hello_app,[]}},
       {env,[]},
       {modules,[hello_app,hello_server,hello_sup]}]}.
    

This is the configuration file that tells Erlang how to start our application. It's just a tuple. The first element is the atom _application_, the second the application name and the thrid a list of key-value tuples. If you are not using _rebar_ don't forget the fullstop at the end. Let's briefly look at the key-value pairs.

* **description**: We should change this to "The hello OTP tutorial app"
* **vsn**: Our applications version. It's common to use x.y.z, but for now, _rebar_'s default is fine.
* **registered**: Is a list of names our app registeres itself under. This is typically only the root supervisors name. In our case we should add hello.sup here, but it's not required.
* **applications**: Lists the applications our app depends on. Every app requires _kernel_ and _sdlib_ to be up and running. Try to add sasl if you like and see what happens when you try to start your application.
* **mod**: Tells Erlang which module contains our application behaviour.
* **env**: Is a list of enviromental variables you may wish to pass to your application.
* **modules**: Is the list of modules our application consists of.

It is time to start our application from the Erlang shell.
    
    
    1> application:start(hello).
    ok
    2> whereis(hello_sup).
    <0.37.0>
    3> whereis(hello_server).
    <0.38.0>
    4> hello_server:say_hello().
    Hello
    ok
    5> application:stop(hello).
    
    =INFO REPORT==== 17-Dec-2012::15:37:47 ===
        application: hello
        exited: stopped
        type: temporary
    ok
    6> whereis(hello_sup).
    undefined
    7> whereis(hello_server).
    undefined
    

Great, now we can start and stop our application simply by name. But there's more. Do you remember how SASL's extended reporting became automatically available when we turned our non-OTP server into a _gen_server_? Try to run the shell session above another time, but first start the Application Monitor using appmon:start(), then, once you've started the 'hello' app, click on the hello button to display the process hierarchy. Pretty nifty, don't you think?
