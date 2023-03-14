---
layout: post
title: "Automating account tasks within Active Directory"
date: 2023-03-13 07:55:00 -0000
categories: Scripting
---

# Table of contents

THIS IS A ROUGH DRAFT, POST HAS NOT BEEN FINISHED.

- [Identifying the problem](#Identifying-problem)
- [Defining the principles behind a need for scripting and automation](#defining-principles)
- [Formatting the data](#formatting-data)
- [Program Design](#defining-philosophy)
- [Recording our actions](#logging-function)
- [Reading the data](#reading-data)
-    [A look into cmdlets and their purpose](#defining-cmdlets)
-    [Using Import-CSV](#defining-import-csv)
-    [Processing the data](#processing-data)
-    [Avoiding error states](#avoiding-errors)

## Identifying the problem {#identifying-problem}

Imagine this: you're an admin for a small office, and one of your common tasks is maintaining and creating user accounts.

You may receive an account creation a week, maybe once a month, and a few password resets here and there. This is not an issue, you'll be able to perform your daily tasks and continue your day. Sure, there has been a lot of setup work to do, but these tasks do not take much of your time so they're essentially just like taking a shower or brushing your teeth; You don't really think about it, you just do it and move on.

One day, you get a job at a large corporation, and now you get anywhere between 10-30 account creations, on top of having a ton more tasks to keep track of. What do you do? Simply get the computer to do your job.

**This post is not intended to show you instructions on how to solve this specific problem, but instead how to work through problems, as it is a very valuable skill.**

## Defining the principles behind a need for scripting and automation {#defining-principles}

For this example, we're going to establish a common procedure, and it looks like this:

1. You receive an email from HR containing a list of people. This list contains their name, employee ID, phone number and job description
2. You validate that the email does come from HR
3. You begin by creating tickets in your ITSM with each employee's information, and keep track of your work by creating additional tickets for each user.
4. You open your Active Directory Users and Computers window, and create a new account in the desired OU
5. You manually add all of the information into the account from each ticket.
6. Finally, you email HR with a list of the employees, and you provide the initial password to the user's manager, also listed on the email from HR.

You may notice a few issues with this approach, here's just a few from my interpretation of it:

- You have to copy and paste a lot of the information from the email
- There isn't a specific format for the information. The HR representative could technically include the information in separate emails, or in a single big email
- There is a lot of repetition, artificially inflating the duration of each account creation task
- You have to come up with passwords manually

There's likely more issues with the current approach than is initially visible, and you would find out over time what these issues specifically are. We are certain of one thing though, and it would be that this approach is not efficient.

Let's come up with a solution that makes sense, in order to reduce our workload so we can focus on other things that are much more complicated. First, we need a common format for the HR representative to submit the information. This would narrow down where we can find the information, and would create a standardized template for the data we're receiving.

After all, the IT field is all about collecting, organizing, documenting, securing, and processing data.

Let's also establish a plan of action to implement a script.

1. Format the data
2. Read the data
3. Enter the data into the ticketing system
4. Enter the data into Active Directory
5. Create an email and send it to the manager.

## Formatting the data {#formatting-data}

We need a format that is easy for the user to understand, but that would still allow us to process the data quickly. In this instance, we're going to make use of Excel. We will create a template and send it to HR as a form they would fill out whenever they have new hires, and it will look like this:

Employee ID | First Name | Last Name | Phone Number | Job Description | Manager

We will make sure this file is in **CSV** format in order to simplify our lives. 

## How can we design a working program? {#defining-philosophy}

To avoid future headaches, we need to first draft out a plan of action for our script. The core concept is that we want to take some data, pass it through a number of functions, then return a desired value.

For our account creation script, we want to pass account information, and get the account, the ticket number, and a pre-formatted email as a result.

Anything that is going to be re-used at some later point in our code we want to turn into a function, and we want to define these functions early on, even if we don't know the implementation aspects just yet.

In this case, we'd like the following:

- A function that records any actions performed by the program, as well as any errors and other useful information.
- A function that reads and returns the data we want, in the format we want. This function will need to take a CSV file and convert it to a format that we can read in code. We'll call this one Read-AccountForm
- A function that creates the ticket in our ITSM portal. 
- A function that creates the account in Active Directory with the data that Read-AccountForm returns
- A function that creates the formatted email and sends it. 

## Recording our actions {#logging-function}

If you've had any software related problem, at some point you've encountered an error, sometimes with a lot of detail. We're going to need that error, as well as what was happening within the program when the error occurred. This will be recorded in a log file. PowerShell does not have integrated logging capabilities, outside of the windows event log, but we want a simple to read file that we can open quickly, so we will not be using the event log.

Why do we need to have an account creation script have logs? You might ask, however, this is a core design philosophy that will help you out in the long run. If your program is going to be used more than once, you're going to want to have the ability to troubleshoot, especially if it will be used by other users. And while the actions that the script executes in the other programs (ITSM, Outlook, Active Directory) are logged, we still want to have a quick glance at what OUR program is doing

So let's design this logging function!

We will need to gather the Date and time of execution, the type of information displayed (to distinguish info messages over potential errors) as well as a message

```powershell

<#

Writes to a log file in the current filepath called ADCreationLogs.
Each line in this file will contain the following format:

DATE TIME   [ACTION]     MESSAGE

ACTION meaning the following values: ERROR, WARNING, INFO, SUCCESS


#>

Function Write-Logfile {
    Param(
    [Parameter(mandatory=$true)]
    [string]
    $Type,
    [Parameter(mandatory=$true)]
    [string]
    $Message
    )

    $LogFile = "$currentloc\ADCreationLogs.txt"
    $Date = Get-Date -Format "yyyy-MM-dd"
    $Time = Get-Date -Format "HH:mm:ss"

    if (!(Test-Path "$currentloc\ADCreationLogs.txt")) {
        New-Item -path $currentloc -name "ADCreationLogs.txt" -type "file"
        Write-Host "Created ADCreationLogs file"
    } 
    else {
        Write-Output "$($Date) $($Time)    [$Type]    $Message" | Out-File $LogFile -Append
        Write-Host "$($Date) $($Time)    [$Type]    $Message"
    }

    
}

```

Now, this does it for any information we want to specify. We can now specify `Write-Logfile $type $message` in order to record something in our log file.

For instance, once the account is created, we want to provide a detail of that in the log. We could achieve this by using our function from earlier:

`Write-Logfile -type "INFO" -message "Created AD account for employee named $name under id $employeeID"`

But we're still missing the errors and other output sent to the console. To capture those, we can use [Start-Transcript](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript?view=powershell-7.3) and [Stop-Transcript](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/stop-transcript?view=powershell-7.3)

At the beginning of our code, we would execute `Start-Transcript -Path "$currentpath\nameoflog.txt" -Append`

The -Append option simply signals that we want to add to the file, as opposed to creating a new one. This is useful if you'd like to have a single log file. In our case, we want to be able to filter a single file, as opposed to different ones. Let's say we're told by HR that one of the accounts was created under the wrong name. We can simply use a find function in Notepad++ with the single file open, and find out when it was created as well as with what data. This will help us troubleshoot and identify if it was our error, or user error.

Then, when program execution stops, we call `Stop-Transcript`

## Reading the data {#reading-data}

From now on, we will be moving onto **PowerShell**. Powershell is a scripting language created by Microsoft that is very powerful and can automate most of, if not all of your daily tasks as an admin. We will also discuss the philosophy behind designing a program like this, which is the entire aim of this post.

### A look into cmdlets and their purpose {#defining-cmdlets}

To first understand how we'll write this script, let's take a look at **cmdlets**

A cmdlet is the PowerShell version of a command and they are built into every PowerShell version. For example, Write-Host is a simple command. All it does is print something to the screen.

Cmdlets are written in the same format: **Verb-Target**

In the case of Write-Host, it simply **writes** to the "host" which in this case is the terminal window that's currently open, if you were to launch it in a new powershell window. If you wanted to take a look at all available powershell commands, you can use **Get-Command**

![Get-Command output](images/Get-Command-Output.png)

Also, you can get information on a specific command by using **Get-Help** or **help** for short.

![Get-Help output](images/Get-Help-Output.png)

These two commands will be the basis behind most of our research within powershell itself. Alternatively, you can access [Microsoft documentation](https://learn.microsoft.com/en-us/powershell/) as another source of information.

## Import-CSV and its purpose {#defining-import-csv}

The first important concept to get down is to test. Test everything, take a deeper look at how data is returned and what you can do with the data.
For this purpose, we're going to create a basic CSV file containing the following data:

ID   | First Name |	Last Name |	Phone Number |	Job Description     |	Manager Email
1000 |	John	  | Johnson   |	123456       |	People's Manager	| info@fakemail.com
1001 |	Bob	      |	Boots     | 7891011      |	Squirrel Counter	| info@fakemail.com
1002 |	Dr        |	Phil      | 12131415     |	Gaming Specialist	| info@fakemail.com

If we want to read and store this data, we may use `Import-CSV` to convert the CSV file contents into a powershell variable. Powershell variables have their own unique properties if they're stored as an object. In this case, we're simply converting the contents of the CSV file into a powershell object of type System.Array

What this means is that we have a very easy to access table with columns and rows. We can loop through each row and find every property to use for our ticket, account and email creation.

Running `Import-CSV .\accounts.csv` (Note: .\ just means current directory) will give us the following output:

```
ID              : 1000
First Name      : John
Last Name       : Johnson
Phone Number    : 123456
Job Description : People's Manager
Manager Email   : info@fakemail.com

ID              : 1001
First Name      : Bob
Last Name       : Boots
Phone Number    : 7891011
Job Description : Squirrel Counter
Manager Email   : info@fakemail.com

ID              : 1002
First Name      : Dr
Last Name       : Phil
Phone Number    : 12131415
Job Description : Gaming Specialist
Manager Email   : info@fakemail.com
```

Alright, that's perfect. Now that we know we can get the data in a readable format, let's think about design for a second.

We're going to be reading (and storing) many CSV files over the course of our week. So we need to define a few things to make adjustments easy to make. Ideally, we'd like to not overengineer the program, while at the same time having enough complexity to scale should the need arise (For example, we have to start populating the Description field).

Let's go over a potential approach, do keep in mind there's many different ways to approach this problem, this is just one of them

## Processing the data {#processing-data}

We can create a for loop to go through each row in the file, then, we would use the specific sets of data to populate AD, our ITSM portal, and the email for the manager. This approach seems to be the simplest, let's evaluate how we would implement it.

If we use `Get-Member -InputObject $file` we will be able to see that there is a method in the powershell object called `Get` since it is an [array](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_arrays?view=powershell-7.3). Looks like this method takes in an `int` which is used to access each member of the array. We can confirm this by calling `$file.get(0)` or alternatively `$file[0]`, either of which will get the first object in the array.

Since the output of `$file[0]` contains ONLY John's information, we can assume the first member of that array would give us the first row. Now we need to find out how this data is stored so we can access it. We will once again use `Get-Member` and the output for that will be:

```
Get-Member -InputObject $file[0]

   TypeName: System.Management.Automation.PSCustomObject

Name            MemberType   Definition
----            ----------   ----------
Equals          Method       bool Equals(System.Object obj)
GetHashCode     Method       int GetHashCode()
GetType         Method       type GetType()
ToString        Method       string ToString()
First Name      NoteProperty string First Name=John
ID              NoteProperty string ID=1000
Job Description NoteProperty string Job Description=People's Manager
Last Name       NoteProperty string Last Name=Johnson
Manager Email   NoteProperty string Manager Email=info@fakemail.com
Phone Number    NoteProperty string Phone Number=123456
```

Looks like `Import-CSV` imports each individual row as a `PSCustomObject` containing properties corresponding to the column names. We will be able to access each property using `Select-Object`. For example, `$file[0] | Select-Object -ExpandProperty ID` would give us `1000` which is the number we want. Since we'll be accesing the same information multiple times, we want to assign each property of the current row into a variable. 

```powershell

$file = Import-CSV .\accounts.csv

ForEach($row in $file) {

    #If we ever need to add another property to the CSV, we can add it here.
    $columnNames = @(
    'ID',
    'First Name',
    'Last Name',
    'Job Description',
    'Phone Number', 
    'Manager Email'
    )
    
    $values = @()

    #We want to loop through each property within $columnNames, and assign the values, in order, to the array $values

    ForEach($val in $columnNames) {$values += $row | Select-Object -ExpandProperty $val}

    #Because of our previous steps! We can actually assign ALL values, in order, directly from the array. We would also need to add a variable here if we were to add one to
    #$columnNames

    $ID, $FirstName, $LastName, $JobDescription, $PhoneNumber, $ManagerEmail = $values

    #Now we can add the logic for the rest of our program, using the variable names
    
    #We could also just use row.'FirstName'

    New-ADUser -EmployeeID $ID -FirstName $FirstName # .....

    #After this, we would create each ticket. Alternatively, we could write the ticket outside of the loop with the entire contents of the CSV, if we wanted one big ticket.
    
}
```

The problem with the above code, is that when adding a new property, you would have to change both `$columnNames` and the variable assignment from `$values`
However, it would work as expected and as we need.

Otherwise, you could forgo variable assignments, and just access each property individually. This is a lot less code, and would be more straightforward.


```powershell

ForEach($row in $file) {

    #In this case, we would just do the following

    #Example: New-ADUser -EmployeeID $row.'ID' -FirstName $row.'First Name'

}

```

It would also be much easier to implement any changes to the CSV file into our code. We would simply access the new property as needed.

## Avoiding error states {#avoiding-errors}

At this point, what we have are pieces. They're individual problems that we've identified from breaking down a big task into small ones. Once we start putting them together, we need to figure out how to make sure that the pieces fit together.

In this case, we need to properly account for the most failstates possible. One such example would be an invalid phone number. Let's say our company only allows phone numbers in the US, now we need to verify that each phone number is in the correct format before pushing the account through. That way, we don't have to individually check each line of the CSV, we would just glance at it to make sure we've got the correct column names.

Since our error message would be shown in the log file, we can address the lack of information on a per-account basis. We would just skip the current item in the for loop if any of our checks fail.

## Conclusions

So, we've got a working design. Now it's time to implement it. From now on, our friend Google can assist with each individual concept. With a solid plan of action, executing every small task should be trivial, since we will be able to search for the information on how to do it.

...

