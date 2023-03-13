---
layout: post
title: "Automating account tasks within Active Directory"
date: 2023-03-13 07:55:00 -0000
categories: Programming Design
---

# Table of contents

[Identifying the problem](#Identifying-problem)
[Defining the principles behind a need for scripting and automation](#defining-principles)
[Formatting the data](#formatting-data)
[Reading the data](#reading-data)
- [### A look into cmdlets and their purpose](#defining-cmdlets)

## Identifying the problem {#identifying-problem}

Imagine this: you're an admin for a small office, and one of your common tasks is maintaining and creating user accounts.

You may receive an account creation a week, maybe once a month, and a few password resets here and there. This is not an issue, you'll be able to perform your daily tasks and continue your day. Sure, there has been a lot of setup work to do, but these tasks do not take much of your time so they're essentially just like taking a shower or brushing your teeth; You don't really think about it, you just do it and move on.

One day, you get a job at a large corporation, and now you get anywhere between 10-30 account creations, on top of having a ton more tasks to keep track of. What do you do? Simply get the computer to do your job.

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

## Reading the data {#reading-data}

From now on, we will be moving onto **PowerShell**. Powershell is a scripting language created by Microsoft that is very powerful and can automate most of, if not all of your daily tasks as an admin. We will also discuss the philosophy behind designing a program like this, which is the entire aim of this post.

### A look into cmdlets and their purpose {#defining-cmdlets}

To first understand how we'll write this script, let's take a look at **cmdlets**


