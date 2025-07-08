# 🐧 Linux Fundamentals and virtualization
 This section documents what I learned during the Linux fundamentals module of my DevOps training. It includes essential command-line operations, user and group management, 
 file editing, service handling, and virtualization using VMs.

## 📂 Basic Linux Operations

### ✅ Topics Covered:
- Navigating directories using `cd`, `ls`, `pwd`
- Managing files and directories using `mkdir`, `touch`, `rm`, `mv`, `cp`
- Viewing file content using `cat`, `less`, `more`, `head`, `tail`
- Understanding file permissions (`chmod`, `chown`)
- Using `man` and `help` for command documentation

  📝 File Editing Tools
Learned Tools:
Nano – Easy terminal-based text editor
Vim – Advanced and powerful editor with command/insert modes

Example: 1> creating a editble file using vim 

![image](https://github.com/user-attachments/assets/fc376362-0402-4cd5-8552-5203755629c6)

   2> Writing a file by enable editing using "i" key
   
   ![image](https://github.com/user-attachments/assets/5d47a602-2f62-46f7-a0df-f53fa8be906f)
   
   3> saving the file usineg kesys ecs and :x and viewing the file using cat
   
   ![image](https://github.com/user-attachments/assets/58be88c4-e707-480a-adc3-991c4d26650f)

🔐 Sudo Privileges & Installing and Deleting Packages/Package management

What I Learned:
The purpose of sudo for privilege escalation
Installing software packages using package managers(apt)
Updating and upgrading the system
Example Commands:

![image](https://github.com/user-attachments/assets/afd74f77-ae0d-4222-9cb3-9f907a7b2e13)

![image](https://github.com/user-attachments/assets/85dcd479-081d-4d1a-89da-36c718000c1e)

 🔧 Service Management
Concepts Practiced:
Starting, stopping, and enabling services
Checking service status

Manually creating a service using systemd
Example:

 ![image](https://github.com/user-attachments/assets/2af1423d-f6b7-47d2-a4ce-181d041625df)
 
![image](https://github.com/user-attachments/assets/3b485f51-ea7d-43fd-9c7f-cad5b7bf31c1)

🛠 Created a Custom Service:

Wrote a .service file to run a custom Python script
Placed it in /usr/lib/systemd/system/
Enabled and started it via systemctl and daemon reload
Example:

![image](https://github.com/user-attachments/assets/61c140dd-0c82-48b8-a1c8-c33dfab67640)

⚙️ Bash Scripting for Automation

What I Learned:
Wrote simple bash scripts to automate repetitive tasks
Used loops, conditionals, and variables
Made scripts executable and reusable

Sample Script:

![image](https://github.com/user-attachments/assets/d784f08a-3363-4952-8e5e-e7faa0f3d595)

Commands Used:
bash
chmod +x calculatormain.sh
./calculatormain.sh

🕒 Scheduled Tasks with Crontab
What I Learned:
Used crontab to schedule scripts
Wrote cron expressions for minute/hour/day scheduling
Command: crontab -e

wrote a crontab file to pull form a repo after every 5 miutes

      */5 * * * * kaushal20 /home/kaushal20/gitpull.sh 


 ![Screenshot 2025-07-08 091737](https://github.com/user-attachments/assets/4a6bade8-ad5c-4336-9a6e-2cde8e2a9706)

      
created a file to store logs of gitpull:


![Screenshot 2025-07-08 091847](https://github.com/user-attachments/assets/c627afac-8a7e-4ed4-a63b-4df57f0be234)

      
Managing User, Group, and File Permissions in Linux

✅ Objective
To grant a newly created user full (rwx) access to a file located at /home/kaushal20.
creatng a new group in root :

![image](https://github.com/user-attachments/assets/831f4347-9185-4613-88cb-9cd1669cf380)

cerating and addig a user in the group :

![image](https://github.com/user-attachments/assets/7683eeff-dcd8-45e1-b985-b4b56e281f76)

Set password for the user:

![image](https://github.com/user-attachments/assets/b6e57388-a09c-4839-806b-20d3cc28b0f2)

Changing group ownereship of the file and set read write execute permission:

![image](https://github.com/user-attachments/assets/e756b710-9a02-4bb2-9fbc-6348b7d24bf3)

Explanation:

7 = rwx for owner

7 = rwx for group

0 = no permission for others

Verify permission:

![image](https://github.com/user-attachments/assets/d6a4a1f9-5e09-485f-84d5-da188142c58c)

💻 Virtualization
🧠 What I Learned:
Installed and used VMware Workstation to create virtual machines

Set up a Linux VM with Ubuntu

Practiced CLI operations inside the VM

![image](https://github.com/user-attachments/assets/3baaae20-edfc-4ea0-b88e-27569aca8c9b)

🔐 Remote Access – SSH and Telnet
Tools Used:
ssh command in Linux terminal

MobaXterm for SSH GUI access

Steps Practiced:
Enabled SSH on the VM

Found the VM’s IP using ip a

Connected using:

![image](https://github.com/user-attachments/assets/caa3d74e-f544-4703-bd41-b58c46806efb)

Accessed VM via MobaXterm for file transfers and GUI-like access














