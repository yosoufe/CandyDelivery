# My  notes

## Environment setup
```bash
conda create --name lerobot python=3.13
conda activate lerobot

cd ../lerobot
python -m pip install -e .[feetech,viz]
```


Finding the port of the robot

```bash
lerobot-find-port

export FOLLOWER_PORT=/dev/ttyACM1 && \
  export LEADER_PORT=/dev/ttyACM0

```

# Setting motor ids

I have already set the motor ids. But the command is `lerobot-setup-motors`. I think I used different command before.
Let's see if we need to continue without doing it with the new tool.

# Calibration

```bash
cd lerobot
python -m pip install -e .[feetech]
sudo usermod -a -G dialout $USER

lerobot-calibrate \
    --robot.type=so100_follower \
    --robot.port=${FOLLOWER_PORT} \
    --robot.id=follower_100

cp /home/yousof/.cache/huggingface/lerobot/calibration/robots/so_follower/follower_100.json .
# or from this repo to cache
cp follower_100.json /home/yousof/.cache/huggingface/lerobot/calibration/robots/so_follower/

lerobot-calibrate \
    --teleop.type=so100_leader \
    --teleop.port=${LEADER_PORT} \
    --teleop.id=leader_100

cp /home/yousof/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/leader_100.json .
# or from this repo to cache
cp leader_100.json /home/yousof/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/
```

Output looks like this
```
NAME            |    MIN |    POS |    MAX
shoulder_pan    |    594 |   1933 |   3407
shoulder_lift   |    752 |    848 |   3236
elbow_flex      |    900 |   3118 |   3126
wrist_flex      |    693 |   2806 |   3198
gripper         |   2032 |   2056 |   3502
Calibration saved to /home/yousof/.cache/huggingface/lerobot/calibration/robots/so_follower/follower_100.json


-------------------------------------------
-------------------------------------------
NAME            |    MIN |    POS |    MAX
shoulder_pan    |    601 |   2009 |   3429
shoulder_lift   |    772 |    859 |   3279
elbow_flex      |    857 |   3113 |   3114
wrist_flex      |    629 |    639 |   3136
gripper         |   2038 |   2055 |   3065
Calibration saved to /home/yousof/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/leader_100.json
INFO 2026-02-06 18:25:21 o_leader.py:156 leader_100 SOLeader disconnected.
```


# Teleoperation
connect the follower first, then the leader

```bash
lerobot-teleoperate \
    --robot.type=so100_follower \
    --robot.port=${FOLLOWER_PORT} \
    --robot.id=follower_100 \
    --teleop.type=so100_leader \
    --teleop.port=${LEADER_PORT} \
    --teleop.id=leader_100
```

# Teleop with camera

Finding the camera and camera's supported parameters.
```bash
lerobot-find-cameras opencv
v4l2-ctl --device=/dev/video4 --list-formats-ext
```

```bash
lerobot-teleoperate \
    --robot.type=so100_follower \
    --robot.port=${FOLLOWER_PORT} \
    --robot.id=follower_100 \
    --robot.cameras="{ handeye: {type: opencv, backend: V4L2, index_or_path: 2, width: 640, height: 480, fps: 30, fourcc: 'MJPG'}}" \
    --teleop.type=so100_leader \
    --teleop.port=${LEADER_PORT} \
    --teleop.id=leader_100 \
    --display_data=true

```


# References

- [so100 docs](https://huggingface.co/docs/lerobot/en/so100?example=Linux&setup_motors=Command&calibrate_follower=Command)