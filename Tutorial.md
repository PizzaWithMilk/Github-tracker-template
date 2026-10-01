Click create new file
<img width="1920" height="1050" alt="zen_UlyOJkQve4" src="https://github.com/user-attachments/assets/f1094d07-2f7c-44a7-9d93-bb0088518c92" />

Then paste `.github/workflows/` in the `name your file` box
<img width="1920" height="1050" alt="zen_jjUwazLuVZ" src="https://github.com/user-attachments/assets/06eb35c1-a8d4-4911-984a-0b8f98b87673" />

Should end up looking like this
<img width="1920" height="1050" alt="zen_iGkYTgUxcL" src="https://github.com/user-attachments/assets/f05cf331-c9a3-4419-a8ec-51fb5b3254e9" />

Then put whatever name you want in the `name your file` box. make sure it ends with .yml
<img width="1920" height="1050" alt="zen_FIbvdTZ7cx" src="https://github.com/user-attachments/assets/9d5e52a4-0996-4f2e-a2d1-446c7eb42a2f" />

Then paste the code from [Repo_Monitor_Template.yml ](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/Repo_Monitor_Template.yml)
<img width="1920" height="1050" alt="zen_I0i3EGVuPH" src="https://github.com/user-attachments/assets/31797fdb-860b-4d99-928b-c22559637e32" />

Then Commit
<img width="1920" height="1050" alt="zen_DsTmBGp42N" src="https://github.com/user-attachments/assets/19f7c354-402c-4631-8637-762e30e3a4a5" />

Then go back to the home page of your repo and Create new file
<img width="1920" height="1050" alt="zen_CCqzL2VRje" src="https://github.com/user-attachments/assets/92655d5b-e269-4d1b-b8ec-754800da3392" />

Then paste `config/repos.yml` in the `name your file` box. should end up looking like this
<img width="1920" height="1050" alt="zen_q1b2ATR6RC" src="https://github.com/user-attachments/assets/8b156a2b-53fd-4782-9a7a-ceb0aa0f5299" />

Then paste the code from [repos.yml](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/repos.yml)
<img width="1920" height="1050" alt="zen_xh4Jwh3DmN" src="https://github.com/user-attachments/assets/8451236d-4825-4c98-9cdb-27ca693c5dc3" />

Then do whatever format you want under `repos:` you can use the examples for guidance. Look at [README](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/README.md#Global-settings) if you want to see what each field does
<img width="1920" height="1050" alt="zen_Rmg2WxXNcj" src="https://github.com/user-attachments/assets/1b7ad102-2341-48ea-83b5-a86aa6028713" />

Then commit
<img width="1920" height="1050" alt="zen_uXVuILxATp" src="https://github.com/user-attachments/assets/c4c79b5d-7a3a-4a5b-85f0-2ef4ff787b19" />

Now you're all done. To make sure it works, Go to your repo's Actions tab. Click on the workflow's name in the list on the left. Click the Run workflow button. 
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/9953ddd0-5410-4e79-88d5-59ceb4402d08" />

Then you should see that it is successful. If it fails then scroll back up and look at how i formatted my repos file and compare it to yours. 
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/a1f080a4-922d-47cd-880f-a2848f9226d6" />

# If you want the monitor to run every 5 minutes without github delaying the scheduled runs by hours then this bottom half is for you.

Now for the external cron. go to https://cron-job.org and sign up, activate your account and then sign in. Once you've signed in you should be at the dashboard.
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/8c2d663b-9411-4f94-b910-1d15ad1d321b" />

On the left side click on `Cronjobs` then click `create cronjob` on the top right.
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/a65c05a1-bd71-4449-948d-f0f1ebb2dd13" />

You should be here
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/dd934c28-3ae7-40f4-8834-a2f393326fcc" />

Name what ever you want in title then paste `https://api.github.com/repos/OWNER/REPO/actions/workflows/FILENAME.yml/dispatches` in the url box. Sorry but you'll have to fill in the url yourself. Keep in mind that this is for your repo with the monitor. only parts you should fill in are `OWNER/REPO/FILENAME`. then put the Execution schedule to every 5 minutes. Don't close this page.
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/e51250f1-3fdf-41ce-8d09-0ff7f603d2f2" />

Now you're going to need a github token. go to https://github.com/settings/personal-access-tokens then click `Generate new token`. Once you've done that you should be here.
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/898ab49e-0c05-4cb0-bf5d-d8c82a4eff06" />

Name the token whatever you want. then set the Expiration to `no expiration`. then in Repository access select `Only select repositories` then select your repo with the monitor in it. Scroll down and click `add permissions` then select both `Actions` and `Contents` then put the access of Actions to `Read and write` and leave `Contents` as `Read-Only`. then click `Generate Token`. Save the token somewhere so you can use it since you'll need it for the next step.

<img width="628" height="1003" alt="image" src="https://github.com/user-attachments/assets/ad4b043f-b037-4fcb-b979-57eddb42887f" />

Now back to the cron page and click `advanced` at the top. you should be here
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/3272e61e-77a6-417f-b586-08e1a320eca8" />

now add 4 headers. in the first header put `Authorization` in the key box. in the second header put `Accept` in the key box. in the third header put `Content-Type` in the key box. in the forth head put `X-GitHub-Api-Version` in the key box. now for the values. in the first header value box put `Bearer YOUR_GITHUB_TOKEN`. replace `YOUR_GITHUB_TOKEN` with the token generated earlier. in the second header value box put `application/vnd.github+json`. in the third header value box put `application/json`. in the forth header value box put `2022-11-28`. now in the `advanced` section under `headers` set the `Request Method` as `POST` then in the `request body` put `{"ref":"main"}` now do a test run. in the action tab of your repo you should see that it ran and everything was successful. then click create on the cron page.
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/193c9933-2cf8-42a0-8ea2-0fe28f971760" />





