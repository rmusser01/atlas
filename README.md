# Atlas
## Table of Contents
- [Intro](#intro)
- [Installing the Agent](#install)
- [Understanding how Atlas Works](#understanding)
- [Extending Atlas](#extending)


### Introduction/What is this Project?
- This repo is the homepage for the [Mythic-C2](https://github.com/its-a-feature/Mythic/) Atlas agent.
- Atlas is a .NET 3.5 and .NET 4.0 compatible agent that is designed as an initial payload offering only the bare essentials before loading upa more fully-featured agent.
- The current version of Atlas supports Mythic `<FIXME>` and will be kept up-to-date as necessary to ensure clean operation with the latest stable release of Mythic-C2.
- It is not compatible with Mythic `<FIXME>` and below.


### How to install Atlas into Mythic<a name="install"></a>
- **Installing from Mythic C2 Host**
  - You can use the `mythic-cli` binary to install Atlas in one of three ways:
    * `sudo ./mythic-cli install github https://github.com/user/repo` to install the main branch
    * `sudo ./mythic-cli install github https://github.com/user/repo branchname` to install a specific branch of that repo
    * `sudo ./mythic-cli install folder /path/to/local/folder/cloned/from/github` to install from an already cloned down version of an agent repo
- You have three(3) options as to when you can install Atlas as an agent into Mythic-C2:
  1. Mythic is already up and going, then you can run the install script and just direct that agent's containers to start (i.e. `sudo ./mythic-cli payload start agentName` and if that agent has its own special C2 containers, you'll need to start them too via `sudo ./mythic-cli c2 start c2profileName`).
  2. Mythic is already up and going, but you want to minimize your steps, you can just install the agent and run `sudo ./mythic-cli mythic start`. That script will first _stop_ all of your containers, then start everything back up again. This will also bring in the new agent you just installed.
  3. Mythic isn't running, you can install the script and just run `sudo ./mythic-cli mythic start`. 



### Understanding How Atlas Works<a name="understanding"></a>
- **Code Architecture**
- **Execution Flow**
		1. Execution Starts in `Main()` in `Program.cs`
		2. One function is immediately called,
            1. `Utils.GetServers();`
            	* This function's execution chain ends with a list of servers being added to the `Config.Servers` list
       	3. The next step is a compile-time flag seeing whether to validate the TLS certificate on callback.
       	4. Next up is a while loop, that checks for if the `HTTP.CheckIn` function returns false, and if so, generates an int and sleeps for that amount of time.
       		* `while (!Http.CheckIn())` - Evals whether `Http.CheckIn()` returns true.
       	5. After the `HTTP.CheckIn` is eval'd, a `JobList` object is created, from `Messages.JobList`. This sets the internal var `job_count` to 0, and `jobs{}` as an array.
       		* `JobList` definition:
       			* https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Messages.cs#L523
       		* `Messages.JobList JobList = new Messages.JobList`
       		* This creates the List object `jobs`
       			* `public List<Job> jobs { get; set; }`
       		* This object also contains a function called `JobList()`
       			* This function creates a new variable `jobs` with the value of `new List<Job>()`
       	6. Finally, we have a while loop `while (true)`, which performs 3 tasks:
       		1. `Dispatch.Loop(JobList);`
       			* This function, as the name implies, first performs an `if/else` check regarding the current date, and should that be satisfied, continues on to:
       				* Retrieve tasking using `Http.GetTasking(JobList);`
       				* Perform said tasking using: `StartDispatch(JobList);`
       				* Returns the results of task execution to the Mythic server: `Http.PostResponse(JobList);`
       		2. `int Dwell = Utils.GetDwellTime();`
       			* `GetDwellTime()`
       			* Creates a high and low number using `Config.Sleep + (Config.Sleep * (Config.Jitter * 0.01))` and `Random()` to pick a 'random' number for each one.
       				* `int Dwell = random.Next(Convert.ToInt32(Low), Convert.ToInt32(High));`
       			* Then multiplies the result by `1000`, and returns that as the value for `Dwell`
       	   	3. `System.Threading.Thread.Sleep(Dwell);`
                * Self-explanatory, process 'sleeps' for the `Dwell` time value established above.
	- **Understanding Tasking of Atlas**
    		1. Tasking occurs During Step 6 above, specifically with the `Dispatch.Loop(JobList);` function.
    			* https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Dispatch.cs#L8
    			* This function first performs a date check, to ensure it is not being ran outside allowed parameters.
    		2. Next, `Http.GetTasking(JobList)` is called, with the argument passed being the `JobList` object from before.
    			1. The function definition can be seen at the below URL.
	    			* https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Http.cs#L78
    			2. The first two lines establish the string field `action` and int field `tasking_size` as get/set-ters
    				* See: https://stackoverflow.com/questions/5096926/what-is-the-get-set-syntax-in-c
    			3. After those two lines, the next line & function defines a string format to use to format strings from tasking messages.
    				* `public string JsonFormat = @"{{""action"":""{0}"",""tasking_size"":{1}}}";`
    			4. Then the function `ToJson()` is defined.
    				1. This function by default takes two arguments, as seen in its definition: `GetTasking` & `get_tasking`
    				2. We can next see that it returns three(3) objects as string values.
    					1. `get_tasking.JsonFormat`
    						* `<Unsure>` This line converts the `get_tasking` object to JSON formatting.
    					2. `JavaScriptStringEncode(get_tasking.action)`
	    					* This line encodes the string stored in `get_tasking.action`.
	    					* https://docs.microsoft.com/en-us/dotnet/api/system.web.httputility.javascriptstringencode?view=net-6.0
    					3. `JavaScriptStringEncode(get_tasking.tasking_size.ToString())`
    						* This line also encodes a string, however, first, the value stored in `get_tasking.tasking_size` is converted to a string object, so that it can be properly encoded as a string.
    		3. Then `StartDispatch(JobList)`, which has the definition line of `public static bool StartDispatch (Messages.JobList JobList)`
    			* https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Dispatch.cs#L19
    			* This function iterates through the `JobList` object, checking to see if any entries match any of the available commands.
    				* If so, the matching code is ran, and the matching `Job` from `JobList` (`JobList.jobs`) is modified to reflect the status of the task.
    		4. Finally, `Http.PostResponse(JobList);`
    			* https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Http.cs#L230
    			* This function is a bit bigger than the others, with the definition line being: 
    				* `public static bool PostResponse(Messages.JobList JobList)`
    			* It first opens a try/catch block, and tries the following:
					* `Messages.PostResponse PostResponse = new Messages.PostResponse`
						* `PostResponse` definition: https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Messages.cs#L200
						* <in-Progress>
    			* Next, it proceeds to iterate over each `job` item in the `JobList.jobs`
    				* `foreach (Messages.Job Job in JobList.jobs)`
	    				* `JobList` Object definition: https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Messages.cs#L523
    					* `Job` object definition: https://github.com/rmusser01/atlas/blob/037349cc85fc362adcb02e6655c530072acd5aa8/Payload_Type/atlas/agent_code/Messages.cs#L534
  
### Extending Atlas's Features & Functionality<a name="extension"></a>
* The agent has `mythic_payloadtype_container==0.0.44` PyPi package installed and reports to Mythic as version "8".
- **101**
    * F
