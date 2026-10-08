# TShark Challenge I : Teamwork

**Difficulty:** 🟢 Easy · **Room:** [TryHackMe ↗](https://tryhackme.com/room/tsharkchallengesone)

You’ve discovered a misconfigured internal automation pipeline running on a Linux server. The system processes recon scripts, development backups, monitoring jobs, and deployment tasks across multiple users. Each stage of the pipeline relies too heavily on the previous one. By abusing these trust boundaries, you must move laterally through the system.

Your objective is to escalate from anonymous access all the way through:

`anonymous access → recon_user → dev_user → monitor_user → ops_user → root`