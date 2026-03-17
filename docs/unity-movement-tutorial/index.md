# Implementing a Character Movement System in Unity

**Technical Writing Sample — Patricia Lo**
Prepared for Technical Writer Applications

---

**Author:** Patricia Lo

**Date:** Jan 2026

**Audience:** Intermediate developers familiar with basic C# scripting and Unity editor navigation

**Documentation Type:** Learning Guide · Real-Time Interactive Systems

---

## Summary

This tutorial explains how to implement player movement using Unity's Character Controller component. It covers the movement pipeline from raw input to final transform updates, including input handling with Unity's Input System, frame-rate-independent motion, jumping and gravity, and third-person camera integration.

By the end of this tutorial, you will have a working character movement system with grounded movement, jumping, and camera-relative directional control.

---

## Prerequisites

| Requirement | Version | Purpose |
|---|---|---|
| Unity Editor | 2023 LTS or later | Game engine and scene editing |
| Input System package | 1.7+ | Modern input handling (`UnityEngine.InputSystem`) |
| C# knowledge | Intermediate | Scripting components and MonoBehaviour lifecycle |

You should be comfortable with GameObjects, components, the Inspector panel, and basic `MonoBehaviour` methods (`Update`, `Start`).

---

## Concept Overview

Before writing any code, it helps to understand the movement pipeline — the sequence of operations that runs every frame to translate player input into on-screen motion.

### Movement Pipeline

```
Player Input (WASD / Gamepad stick)
   ↓
Movement Vector Calculation
   ↓
Gravity & Jump Force Applied
   ↓
Character Controller Collision Resolution
   ↓
Final Transform Update
```

**Why this order matters:** The Character Controller's `.Move()` method handles collision detection internally. By computing the full movement vector (horizontal + vertical) *before* calling `.Move()`, you let the controller resolve all collisions in a single pass, avoiding physics instability from multiple move calls per frame.

---

## Step 1 — Unity Input System Setup

### Action

Install the Input System package via **Window > Package Manager**, then create an Input Actions asset.

### Why

The legacy `Input.GetAxis()` API is being phased out. The new Input System decouples input bindings from game logic, making it easier to support multiple control schemes (keyboard, gamepad, touchscreen) without changing movement code.

### Implementation

Create a `PlayerInput` actions asset with a `Move` action (Value, Vector2) and a `Jump` action (Button):

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerMovement : MonoBehaviour
{
    private PlayerInputActions inputActions;
    private Vector2 moveInput;

    private void Awake()
    {
        inputActions = new PlayerInputActions();
    }

    private void OnEnable()
    {
        inputActions.Player.Enable();
        inputActions.Player.Jump.performed += OnJump;
    }

    private void OnDisable()
    {
        inputActions.Player.Jump.performed -= OnJump;
        inputActions.Player.Disable();
    }

    private void Update()
    {
        moveInput = inputActions.Player.Move.ReadValue<Vector2>();
    }
}
```

### Result

Input is now captured as a `Vector2` each frame without being coupled to specific keys. Rebinding controls later requires no code changes.

---

## Step 2 — Character Controller Setup

### Action

Add a `CharacterController` component to your player GameObject via the Inspector.

### Why

The `CharacterController` provides built-in collision handling without rigidbody physics. Unlike a `Rigidbody` + `Collider` setup, it won't slide on slopes unexpectedly or get pushed by other physics objects — behavior you typically don't want for a player character.

### Implementation

Reference the controller in your script:

```csharp
[RequireComponent(typeof(CharacterController))]
public class PlayerMovement : MonoBehaviour
{
    [SerializeField] private float moveSpeed = 5f;

    private CharacterController controller;

    private void Awake()
    {
        controller = GetComponent<CharacterController>();
    }
}
```

### Result

The player GameObject can now be moved programmatically via `controller.Move()`, with collision detection handled automatically.

---

## Step 3 — Movement Logic

### Action

Convert the 2D input vector into a 3D world-space movement direction and apply it through the Character Controller.

### Why

`moveInput` is a `Vector2` (X = horizontal, Y = forward/back), but the game world uses `Vector3`. You need to map input axes to world-space directions and multiply by `Time.deltaTime` so movement speed is consistent regardless of frame rate.

### Implementation

```csharp
private void Update()
{
    moveInput = inputActions.Player.Move.ReadValue<Vector2>();

    Vector3 move = transform.right * moveInput.x + transform.forward * moveInput.y;
    controller.Move(move * moveSpeed * Time.deltaTime);
}
```

### Result

The character moves relative to its facing direction at a consistent speed. At 30 FPS or 144 FPS, the character covers the same distance per second.

---

## Step 4 — Jumping and Gravity

### Action

Add vertical velocity tracking with gravity and a jump impulse.

### Why

The Character Controller does not apply gravity automatically — it only resolves collisions. Without manually applying a downward force each frame, the character will float in the air after walking off a ledge.

### Implementation

```csharp
[SerializeField] private float jumpHeight = 1.2f;
[SerializeField] private float gravity = -19.62f;

private Vector3 velocity;

private void Update()
{
    if (controller.isGrounded && velocity.y < 0)
    {
        velocity.y = -2f;
    }

    // Horizontal movement
    moveInput = inputActions.Player.Move.ReadValue<Vector2>();
    Vector3 move = transform.right * moveInput.x + transform.forward * moveInput.y;
    controller.Move(move * moveSpeed * Time.deltaTime);

    // Jumping
    if (controller.isGrounded && jumpRequested)
    {
        velocity.y = Mathf.Sqrt(jumpHeight * -2f * gravity);
        jumpRequested = false;
    }

    // Gravity
    velocity.y += gravity * Time.deltaTime;
    controller.Move(velocity * Time.deltaTime);
}

private void OnJump(InputAction.CallbackContext context)
{
    jumpRequested = true;
}
```

!!! note "Why -2f instead of 0?"
    Setting `velocity.y = -2f` when grounded (instead of `0`) keeps the controller firmly pressed against the ground. This ensures `controller.isGrounded` returns `true` consistently, avoiding a common bug where the character "flickers" between grounded and airborne states.

### Result

The character responds to gravity, falls off edges naturally, and can jump to a configurable height.

---

## Step 5 — Camera Integration

### Action

Set up a third-person camera that follows the player and drives movement direction.

### Why

Without camera-relative movement, pressing "W" always moves the character along the world's Z-axis — confusing when the camera is rotated. Players expect "forward" to mean "away from the camera."

### Implementation

Replace `transform.right` / `transform.forward` with the camera's orientation:

```csharp
[SerializeField] private Transform cameraTransform;

private void Update()
{
    moveInput = inputActions.Player.Move.ReadValue<Vector2>();

    Vector3 forward = cameraTransform.forward;
    Vector3 right = cameraTransform.right;

    // Flatten to horizontal plane
    forward.y = 0f;
    right.y = 0f;
    forward.Normalize();
    right.Normalize();

    Vector3 move = right * moveInput.x + forward * moveInput.y;
    controller.Move(move * moveSpeed * Time.deltaTime);

    // Rotate character to face movement direction
    if (move.sqrMagnitude > 0.01f)
    {
        Quaternion targetRotation = Quaternion.LookRotation(move);
        transform.rotation = Quaternion.Slerp(
            transform.rotation, targetRotation, 10f * Time.deltaTime
        );
    }

    // ... gravity and jump logic unchanged
}
```

!!! info "Why flatten the camera vectors?"
    The camera may be angled downward. Zeroing out the Y component and re-normalizing ensures movement stays on the horizontal plane, preventing the character from diving into the ground when the camera looks down.

### Result

Movement is now camera-relative. The character faces the direction it's moving, and controls feel intuitive regardless of camera angle.

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| Movement speed varies by frame rate | Missing `Time.deltaTime` in `Move()` call | Multiply all per-frame motion by `Time.deltaTime` |
| Character floats / doesn't fall | No gravity applied to vertical velocity | Apply `gravity * Time.deltaTime` to `velocity.y` every frame |
| `isGrounded` flickers between true/false | `velocity.y` resets to `0` instead of a small negative value | Set `velocity.y = -2f` when grounded |
| "Forward" doesn't match camera direction | Using `transform.forward` instead of camera forward | Derive movement direction from `Camera.main.transform` |
| Character tilts into the ground | Camera forward vector has a Y component | Zero out Y and re-normalize before using as a direction |

---

## Performance Considerations

- **Separate input from movement logic.** Reading input in `Update()` and applying physics in `FixedUpdate()` can prevent input lag in physics-heavy scenes.
- **Avoid heavy calculations inside `Update()`.** Cache component references in `Awake()` and reuse vectors instead of allocating new ones each frame.
- **Keep movement values configurable.** Expose `moveSpeed`, `jumpHeight`, and `gravity` as `[SerializeField]` fields so designers can tune them in the Inspector without code changes.

---

## Related Topics

- **Camera follow systems** — Cinemachine FreeLook for orbital third-person cameras
- **Animation blending** — Connecting movement speed to Animator blend trees
- **Physics-based movement** — When to use `Rigidbody.AddForce()` instead of `CharacterController.Move()`
- **Networked movement** — Client-side prediction and server reconciliation for multiplayer
