---
title: "How I Estimate My Tasks"
author: ""
type: ""
date: 2021-05-08T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
When a task is simple, estimating isn’t a problem but when the task is complex problems arise. Sometimes I have to estimate complex tasks to measure the sprint time or other activities.

Occasionally my boss asks me why I estimated a task for N hours/days so I have an explanation. In my opinion, the best explanation is a sub-tasks list where each sub-task has a lead time.

My requirements for a task estimation:
* total task lead time is no more than 18h (2d)
* each sub-task lead time is no more than 2h

In my experience, if a task is estimated in more than 18 hours, the task is unclear and inaccurate. It will never be completed on time.

I estimate complex tasks in several steps.

## Step 1. Requirements Investigation
When I have a requirements document, I start by investigating it. Then I discuss the document with the placeholders to avoid misunderstanding. Unless I have the requirements document I discuss the requirements with placeholders and write a short requirements document.

At the end of this step, I have a strong understanding of the requirements for the task.

## Step 2. Code Investigation
I go over a project code to find the places where I will make changes to accomplish the task. I investigate these places on future changes and consider refactoring needs.

If needed, I come up with new abstractions or break down current abstractions into small ones if they will be large or unclear after implementing the task.

At the end of this step, I understand how and where I will make changes in the code.

## Step 3. Planning
During previous steps, I write down each step of task implementation. Thus, I build up a plan or TODO list. Then I estimate each step separately. 

I include in estimation a time to agree on the task and deploy the one.

According to my requirements, sub-task time has to be less than 2 hours. If they are more than 2 hours I decompose them into small ones. Then I count the total task implementation time and if the task time is more than 16 hours I break down this task into small ones.

Sometimes I discuss the plan with placeholders if I don’t confident about the task. When I implement a task for another team I agree on the API with members of that team.

## Example
Task requirements: add a new API for exposing tags to the frontend.
The plan:
1. Design an API - 30m
2. Write documentation about the API - 30m
3. Agree on the API with the frontend team - 1h
4. Refactor a code in service - 3h
5. Add a new method to repository interface - 10m
6. Add new SQL query - 30m
7. Write tests for the repository - 30m
8. Add a new method to a controller - 20m
9. Write a test for the new controller method - 30m
10. Test the API on stage server - 1h
11. Deploy and verify the API in production - 1h

Lead time: 1d 1h or 9h

## Conclusion
As a result of these three steps, I will end up with a plan (TODO list) for the task. Each step in the plan will have a lead time. I can count the time of all steps and get the total lead time for the task. Also, if someone asks me why I estimated a task for N hours I show him the plan.
