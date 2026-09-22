Click create new file
<img width="1920" height="1050" alt="zen_UlyOJkQve4" src="https://github.com/user-attachments/assets/f1094d07-2f7c-44a7-9d93-bb0088518c92" />

Then paste `.github/workflows/` in the `name your file` box
<img width="1920" height="1050" alt="zen_jjUwazLuVZ" src="https://github.com/user-attachments/assets/06eb35c1-a8d4-4911-984a-0b8f98b87673" />

Should end up looking like this
<img width="1920" height="1050" alt="zen_iGkYTgUxcL" src="https://github.com/user-attachments/assets/f05cf331-c9a3-4419-a8ec-51fb5b3254e9" />

Then put whatever name you want in the `name your file` box. make sure it ends with .yml
<img width="1920" height="1050" alt="zen_FIbvdTZ7cx" src="https://github.com/user-attachments/assets/9d5e52a4-0996-4f2e-a2d1-446c7eb42a2f" />

Then paste the code from either [Repo_Release_Tracker_Template.yml](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/Repo_Release_Tracker_Template.yml), [Repo_Release_Tracker_Template_Forum_Channel.yml](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/Repo_Release_Tracker_Template_Forum_Channel.yml), [Repo_Commit_Tracker_Template.yml](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/Repo_Commit_Tracker_Template.yml) or [Repo_Commit_Tracker_Template_Forum_Channel.yml](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/Repo_Commit_Tracker_Template_Forum_Channel.yml) 
<img width="1920" height="1050" alt="zen_sPOhh3WJ3L" src="https://github.com/user-attachments/assets/6b3adc97-4e2c-433a-b1ef-a131aa23ae78" />

Then Commit
<img width="1920" height="1050" alt="zen_69U5QXUFT8" src="https://github.com/user-attachments/assets/af426d9b-adbc-404b-b0c0-613e195fcf06" />

Then go back to the home page of your repo and Create new file
<img width="1920" height="1050" alt="zen_CCqzL2VRje" src="https://github.com/user-attachments/assets/92655d5b-e269-4d1b-b8ec-754800da3392" />

Then paste `config/repos.yml` in the `name your file` box. should end up looking like this
<img width="1920" height="1050" alt="zen_q1b2ATR6RC" src="https://github.com/user-attachments/assets/8b156a2b-53fd-4782-9a7a-ceb0aa0f5299" />

Then paste the code from [repos.yml](https://github.com/PizzaWithMilk/Github-tracker-template/blob/main/repos.yml)
<img width="1920" height="1050" alt="zen_YNWJH9jOcx" src="https://github.com/user-attachments/assets/35e9cda1-2134-4096-aacb-43223916d009" />

Then do whatever format you want under `repos:` you can use the examples for guidance.
<img width="1920" height="1050" alt="zen_gwixpfipx4" src="https://github.com/user-attachments/assets/fc0bb18d-4fc3-45d7-83b8-e2a5a4778bea" />

Then commit
<img width="1920" height="1050" alt="zen_BeGdpgqNvT" src="https://github.com/user-attachments/assets/59eef54c-02c2-4b2d-9ff7-cf112e77e06b" />

Then go to Settings → Secrets and variables → Actions → New repository secret.
<img width="1920" height="1050" alt="zen_AJAGoWYRZS" src="https://github.com/user-attachments/assets/e743cf9d-2be6-4677-b4a1-62c114e1dece" />

Then name the secret `DISCORD_WEBHOOK`
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/f4327275-f869-4cf2-a461-8d0df894dd51" />

Then In Discord, go to the channel you want notifications posted into. Open its settings by click the gear icon, or right-click → Edit Channel then go to Integrations → Webhooks → New Webhook Give it any name you like, then click Copy Webhook URL and paste it into the secret box
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/5bd3fd96-003b-4a43-8504-ecf5680ebbd3" />

Then click add secret
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/b7e302e9-58e8-4733-82e0-86f8c90a71db" />

Now you're all done. To make sure it works, Go to your repo's Actions tab. Click on the workflow's name in the list on the left. Click the Run workflow button. 
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/9953ddd0-5410-4e79-88d5-59ceb4402d08" />

Then you should see that it is successful. If it fails then scroll back up and look at how i formatted my repos file and compare it to yours. 
<img width="1920" height="1050" alt="image" src="https://github.com/user-attachments/assets/a1f080a4-922d-47cd-880f-a2848f9226d6" />







