11/19/2026

Policy:
The first policy I created for the xlerobot was for the right arm to start at "home" position folded up, reach out towards the table while opening its gripper, picking up a block about 4 inches, moving towards the X piece of tape, lowering the block until it's resting on the X, releasing the gripper, then moving the arm back to "home" position while closing the gripper.  I had 5 different starting places for the block denoted by a piece of tape.

Continuation from 2026-11-03:
After fixing the tiny standard deviation blowing up on policy rollouts by adding the standard deviation floor, it was time to rollout the policy and get the baseline data.  The 20k checkpoint was used with n_action_steps = 100, so the policy would get a new set of 100 actions after 100 actions had taken place, so the policy would take a snapshot from the camera once every ~3.3 seconds and then plan the next 100 actions before looking again.

The policy was run 20 times rotating through the 5 different start positions noting grasp on first try, grasped, placed on the X, placed touching the X and did the robot return home.  Additionally, it was noted how the session ended (tripwire of a motor getting told to move to a position 30 deg different or 30 sec duration ended), and notes on how the run went.  The robot only picked up the block 1/20 runs and it ended 12/20 times by tripping the tripwire.

I wanted to learn more about what was happening when the tripwire was triggered, so I created a script to track goal and present positions of the motors at each tick.  The data revealed that the jumps were happening right at the start of a new 100 action chunk.

In order to smooth the seams, I decided to run the policy using ensembling.  Ensembling has the robot take in a camera image every action and create another 100 action step plan.  Then it takes an average of the plans to choose the next action, weighing the past actions with the new suggested action to keep the actions smooth.  This hopes to remove the 100 action seam jumps.

I tried running it for the first time, and the tripwire tripped immediately.  I had forgotten to take the protective camera cap off before, so I checked that first, but it was off.  I tried running the policy again and it tripped at 0 again.  I tried running the original rollout with the 100 action chunks and it failed too.  It was then, that I noticed that the left robot arm's gripper was open.  I did all the training data with the left arm stationary with the gripper closed, so this was something the policy had never seen.

This is the same how unusual score in action again.  In the demo data, the left gripper always read 1.0, making it a standard deviation of 0, but as noted in 2026-11-03, I raised the standard deviation floor to 1.0.  so the unusual score = (live reading - training average) / standard deviation = (10.2 - 1.0) / 1 = 9.2.  The model only saw scores of 0, so 9.2 is way outside what it had seen before.  Every input feeds every output, so the 9.2 affects every joint the policy controls, including the right arm.

After closing the gripper, I ran the ensembled rollout 10 times.  The number of trips at the seams reduced from 12/20 to 0/10, with each run timing out at the end of the 30 second duration.  The robot also successfully grasped 3/10 runs and placed touching the target 2/10 times, but never placed it fully on the X 0/10.

The ensembled runs also were able to run without slowing down the Hz (~29.6Hz).

Additionally, the robot crashing into the table at the end of the run duration and when hitting the tripwire was corrected by adding a torque on hold for the arm with a torque off for the gripper to release objects or pinches safely.







Some things I want to consider to try to improve results:
1. I realized from the home position, the camera is unable to see position 1 and the dropoff X since they are out of the frame, and can only see half of position 5 since the gripper covers it.
    a. I want to have all positions be visible to the camera from the home position.
    b. I want to try using the overhead depth sensing camera since it has a better view of the table
2. I never taught the robot to sit in the home position after the run was completed since I terminated the recording once it made it home.  This might be why after the robot finished dropping off a block, it goes home but then gets back up again.  I want to learn how to teach the robot that the task is complete and it can move on to the next task.
3. I'm curious if the touch sensors can help the robot determine if it successfully picked up the block (although it should be able to tell from the video that nothing is in its gripper)
4. I want to learn more about how to make policies more robust.  For example, the left arm gripper being open made it trip on tick 0.  Maybe the gripper could close or the policy could ignore the stationary arm and just leave it in whatever position it's in.
