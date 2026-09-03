# LIBERO post-training

`train.yaml` fine-tunes the pretrained Tau0VLA checkpoint on LIBERO while
preserving the checkpoint's unified 40D state/action interface. It selects the
`libero_eef_robot_prompt_ft` data route defined in `data.py`.

## Native LIBERO contract

```text
state  = [eef_xyz(3), eef_axis_angle(3), gripper_qpos(2)]  # 8D
action = [delta_xyz(3), delta_axis_angle(3), gripper(1)]   # 7D
action_horizon = 10
```

The action values already represent EEF deltas, so the action route uses
`abs2relative=False` and does not subtract the current state a second time.

## Checkpoint-aligned 40D representation

The native 3D axis-angle rotation is converted to a 6D rotation
representation. Consequently, each EEF pose changes from
`xyz(3) + axis-angle(3) = 6D` to `xyz(3) + rot6d(6) = 9D`.

Both state and action then use the following model-facing layout:

| 40D slots | Meaning | Active |
| --- | --- | --- |
| `0:3` | EEF xyz or delta xyz | yes |
| `3:9` | EEF rot6d or delta rot6d | yes |
| `9:18` | reserved/padding | no, always zero |
| `18` | left gripper | yes |
| `19:40` | reserved/padding | no, always zero |

The conversion and padding pipeline is:

1. `AxisAngle2Rot6D` converts the 6D EEF pose to 9D.
2. `PadToDim(9, 18)` right-pads the EEF component so that the following
   gripper component is placed at slot `18`.
3. `state_padding_dim=40` and `action_padding_dim=40` pad the assembled
   19D vectors to the checkpoint's 40D input/output dimensions.

For state input, LIBERO's two opposing finger joints are reduced to one
opening value:

```text
gripper = 0.5 * (qpos[0] - qpos[1])
```

The active state/action indices are therefore `0:9` and `18`. In particular,
`use_action_mask_loss: true` excludes all inactive action dimensions from the
flow-matching loss, while `vla_inactive_input_zero: true` keeps those inactive
action dimensions at zero in the flow input. `zero_state_emb: false` keeps
state conditioning enabled. During deployment, predicted rot6d EEF rotations
are converted back to native 3D axis-angle commands before they are returned
to the LIBERO simulator.

## Training

Set the dataset path (or a text manifest containing one dataset path per line)
and launch training:

```bash
export TAU0_LIBERO_DATA=/path/to/libero
bash scripts/train.sh configs/libero/train.yaml \
  --model_name_or_path sii-research/tau-0-vla
```

The selected robot-aware prompt is:

```text
You are controlling a robot.
Robot type: Panda
Control mode: end-effector
Whole-body control: disabled
Task: {instruction}
```

`LiberoRobot` intentionally overrides stale `field_descriptions` found in some
older exports. The eight state values are always interpreted as six EEF values
followed by two gripper values.

Serve the resulting checkpoint with the simulator-only EEF server:

```bash
python -m deploy.libero_server --model outputs/tau0_vla_libero_eef_ft_40_align
```

The server binds to localhost and exposes `POST /act_libero`, `POST
/reset_episode`, and `GET /health`. Requests to `/act_libero` are pickled
dictionaries using the keys documented by `python -m deploy.libero_server
--help`.

The server returns the action horizon recorded by the fine-tuned checkpoint
(10 actions with the default config). A LIBERO evaluator may execute only the
first `replan_steps` actions and then request a fresh chunk.
