# Repository and Lab Workflow

The Git repository is the course source of truth. Weekly commands sometimes
use disposable lab directories to make experiments easy to reset, but a week is
not complete until its reusable source, tests, decisions, and sanitized
evidence are promoted into the repository.

## One active course clone

Run ROS inside the ARM64 Ubuntu 24.04 VM. Use either a Git clone inside the VM
or one VM-mounted clone edited from the Mac, but do not maintain two unsynced
working copies.

The commands in this course assume:

    git clone https://github.com/varun1524/robotics-learning.git \
      "$HOME/robotics-learning"
    cd "$HOME/robotics-learning"
    export COURSE_REPO="$(pwd -P)"
    test -f "$COURSE_REPO/docs/learning-plan.md"

On this curriculum branch, the repository root is also the colcon workspace:

    cd "$COURSE_REPO"
    rosdep install --from-paths src --ignore-src -r -y
    colcon build --base-paths src

Build, install, and log output is ignored. Never source an install directory
from a different clone.

## Tracked versus disposable material

| Material | Required destination |
| --- | --- |
| ROS packages, services, and reusable scripts | src |
| Versioned message contracts | proto |
| Unit, contract, integration, and fault tests | tests |
| Architectural choices and measured limits | decisions |
| Small sanitized results and CSV summaries | evidence/week-NN |
| Large local recordings | bags/week-NN, ignored except manifest |
| Build/runtime databases and temporary lab files | build, install, log, or work, all ignored |

Some assignments create prototypes under HOME/robotics_ws,
HOME/robotics-learning-lab, or HOME/robotics-lab. Treat those paths as scratch.
Before checking the week complete, move or re-create the final implementation
under the destinations above, change its launch/test commands to use
COURSE_REPO, and confirm the scratch directory can be removed without losing
the result.

## Promotion map

| Weeks | Reusable repository result |
| --- | --- |
| 0–2 | ROS graph/comms and motion packages under src; automated checks under tests |
| 3–5 | robot_description, robot_sim, sensor configuration, and replay tests |
| 6–7 | map tools, navigation package/configuration, and fixed evaluation fixtures |
| 8 | mission_runner and behavior-tree tests |
| 9 | canonical_adapter plus telemetry_gateway and their contract tests |
| 10 | command_journal plus replay, duplicate, and capacity tests |
| 11 | map_service plus immutable-version/release fixtures |
| 12 | media_gateway and artifact-transfer integration tests |
| 13–14 | simulator or physical adapter and navigation calibration profile |
| 15 | identity_service tests; no generated keys or credentials |
| 16 | capstone integration tests, runbook, acceptance profile, and evidence manifests |

The package names in an assignment take precedence when they are more
specific. Do not copy generated databases, private keys, bags, or raw media
into source directories.

## Weekly clean-checkout gate

Before marking a week complete:

1. remove dependence on files that exist only in a scratch directory;
2. record dependencies and exact launch/test commands;
3. run the relevant tests from COURSE_REPO;
4. run the pre-commit secret scan;
5. verify every large asset using a recorded SHA-256 digest; and
6. ask another shell or clean clone to reproduce the result.

Minimum repository checks:

    cd "$COURSE_REPO"
    pre-commit run --all-files
    colcon build --base-paths src
    colcon test --base-paths src
    colcon test-result --verbose
    git diff --check
    git status --short

If no ROS package exists yet, the colcon build/test commands may report an
empty workspace in Week 0. All other checks still apply.

## Large assets

Commit a small source fixture when its license and size permit. Otherwise,
commit a manifest with media type, byte length, license, SHA-256 digest, and a
public fetch or deterministic regeneration command. The consumer verifies the
digest before use and fails closed on mismatch.

Private URLs, temporary object credentials, private sensor recordings, and
generated certificates never belong in a public manifest.

[Repository home](../README.md) · [Documentation](README.md) ·
[Weekly assignments](weeks/README.md)
