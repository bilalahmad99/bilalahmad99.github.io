---
layout: default
---

## DevSecOps: Security breach Incident protocol

Security breaches in the tech world are increasing at an alarming rate. If it hasn't happened for you, be prepared for it because as the saying goes *Hope for the best and prepare for the worst*.

![hope for the best, prepare for the worst](../assets/img/hope_for_best.jpeg)

I have had my fair share in my years in the tech world so far. I am listing down a well thought out protocol that we as a team created for any such occupance. So here it goes:

#### Take a deep breath !
Nothing ever good happens when you panic. Stay calm and collected.

![calm](../assets/img/burning_house.gif)

#### Set up response teams
Suggestion is to setup two teams:
A primary team which will be first responders and sit in the same conference hall or meeting room aka command center.
A secondary team which will be ready to take over from primary team at any stage and should be fully in-sync in terms of communication with primary team.

Now if the incident is not at a big scale, you can skip the secondary team.
Here is who needs to be part of primary team though:
Team Lead (the captain of team, also leading the communication with rest of stakeholders who are not part of team)
Transcriber/ Auditor (can be a single person or 2 at max; takes written notes of every action the team is taking with timestamps)
Technical Support members
Legal Support members

#### Snapshot the current state of systems
Create snapshots of bastion hosts, DBs or any current running systems which could be or are compromised

#### Limit the access to infrastructure
Block all access to the Bastions or your primary cloud accounts except for the ones that are in the team. Re-create the access keys of all individuals who are in team.

#### Check Logs
Watch for failed login attempts or unknown logins in any of your critical accounts e.g. Cloud provider account, DNS provider account, mail account etc.

#### Determine source of incident
From the logs or the impacted system try to investigate how the breach could have happened. If its not straight forward to figure out, try to think of all ways any person could reach to the affected system and eliminate the options one by one.

#### Assess the impact
After the attacker has compromised a machine or system, try to asses how much further he can go in the infrastructure using that system. Think of all the worst case scenarios and note down as much impact as possible.

#### Discuss and agree on the actions to be taken
The first priority of the action you agree on is to be quickly contain and minimize the impact.
Assign the action items to individual members of the teams with the due time and check on progress.

#### Post Incident Review
Discussions of what went right and wrong. How this can be avoided in future.
Recommendations to the teams and company on how this can be avoided in future.
Follow-up on long terms action items taken from discussion.


[back](../)