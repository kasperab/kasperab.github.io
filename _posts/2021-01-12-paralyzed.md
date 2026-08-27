---
title: Paralyzed by Fear
description: Experimental extraterrestrial exploration.
category: Game
tags: Godot Solo
web: "https://kasperab.itch.io/paralyzed-by-fear"
code: "https://github.com/kasperab/paralyzed-by-fear"
---

{% include video.html name="paralyzed" %}

The basic idea for this game was that instead of controlling the player character directly you would control their emotions, and then the character would move and act based on those emotions. I also wanted it to be some sort of exploration game with environmental storytelling, and so I decided to make the setting an alien planet with strange and scary vegetation, in order to make the experience feel even more weird and novel. After thinking about what emotions would be relevant in a situation where you find yourself stranded on an alien planet, I settled on fear, curiosity, and aggression (I guess I'm using "emotions" loosely here).

I started by putting these emotions on each corner of a triangle and programming a marker on this triangle the player can control. The marker can be controlled by pressing the arrow keys or by clicking on the triangle. To create some extra feeling for the current emotional state I added a fog overlay image which changes color based on the values of the emotions.

```gd
func _process(delta):
	var fearPressed = fearButton or Input.is_action_pressed("fear")
	var curiosityPressed = curiosityButton or Input.is_action_pressed("curiosity")
	var aggressionPressed = aggressionButton or Input.is_action_pressed("aggression")
	if fearPressed and curiosityPressed and aggressionPressed:
		return
	elif fearPressed and curiosityPressed:
		if aggression <= 0:
			return
		fear += emotionSpeed * delta
		curiosity += emotionSpeed * delta
		aggression -= emotionSpeed * 2 * delta
	#...more of the same...
	elif fearPressed and fear < 1:
		fear += emotionSpeed * 2 * delta
		if curiosity <= 0:
			aggression -= emotionSpeed * 2 * delta
		elif aggression <= 0:
			curiosity -= emotionSpeed * 2 * delta
		else:
			curiosity -= emotionSpeed * delta
			aggression -= emotionSpeed * delta
	#...more of the same...
	updateMarker()

func updateMarker():
	var x = 224 - 224 * fear + 224 * aggression
	var y = 384 - 384 * curiosity
	get_node("../UI/Marker").rect_position = Vector2(x, y)
	var fogColor = get_node("../UI/Fog").get_modulate()
	fogColor.r = aggression * 0.5
	fogColor.g = curiosity * 0.5
	fogColor.b = fear * 0.5
	get_node("../UI/Fog").set_modulate(fogColor)
```

After that I built an alien forest for the player to explore. The player has one collision body acting as vision, and one acting as the physical body. When the player moves around and a tree (or other object in the forest) enters their vision, it is added to either the list of scary or interesting objects, provided the player has not already examined an object of the same type. If the player is on their way to examine an object and that object enters their body, they start examining it. Examining machines (ID less than 10) will give the player scrap, and examining scary objects will kill the player.

```gd
func _on_Vision_enter(body):
	obstacles.append(body)
	if body.name == "InterestingBody":
		for index in range(examinedObjects.size()):
			if body.id == examinedObjects[index]:
				return
		interestingObjects.append(body)
	elif body.name == "ScaryBody":
		scaryObjects.append(body)

func _on_Vision_exit(body):
	obstacles.erase(body)
	if body.name == "InterestingBody":
		interestingObjects.erase(body)
	if body.name == "ScaryBody":
		scaryObjects.erase(body)

func _on_Body_enter(body):
	if body.name == "InterestingBody" and body.id == targetID:
		examinedObjects.append(body.id)
		target = null
		var index = 0
		while index < interestingObjects.size():
			if interestingObjects[index].id == body.id:
				interestingObjects.remove(index)
			else:
				index += 1
		$Astronaut/AnimationPlayer.current_animation = "examine"
		$Astronaut/AnimationPlayer.playback_speed = 1
		$ExamineAudio.play()
		examining = true
		if body.id < 10:
			scrap += 1
	elif body.name == "ScaryBody" and body.id == targetID:
		$Astronaut/AnimationPlayer.current_animation = "examine"
		$Astronaut/AnimationPlayer.playback_speed = 1
		$ExamineAudio.play()
		examining = true
		dead = true
```

The player's behaviour is determined by the current emotional state. Curiosity will lead the player to start moving towards interesting objects in the forest. This is an important part of the game, but too high curiosity will cause the player to move towards scary objects and certain death. If there are no interesting or scary objects in sight the player will simply move towards the center of the world. High enough aggression will cause the player to just stop and shoot. What constitutes a high enough aggression depends on if there are scary objects in sight. If the fear is too high the player will simply stand still, paralyzed. This is where the game gets its name from, and it is also the initial emotional state.

When thinking about how to control a character through emotions instead of directly, I realized that the easiest way to conceptualize this would be to think of it like I'm making AI. For this part of the code I think it would have been better if I had leaned into that more and used a behavoiur tree.

```gd
var position = Vector2(translation.x, translation.z)
if fear > 0.9:
	target = null
	targetID = 0
elif reload <= 0.0 and (aggression > 0.8 or (aggression > 0.5 and scaryObjects.size() > 0)):
	target = null
	targetID = 0
	shoot()
elif target == null and curiosity > 0.1:
	if curiosity > 0.8 and scaryObjects.size() > 0:
		var targetIndex = randi() % scaryObjects.size()
		var targetPosition = scaryObjects[targetIndex].to_global(Vector3())
		target = Vector2(targetPosition.x, targetPosition.z)
		targetID = scaryObjects[targetIndex].id
	elif interestingObjects.size() > 0:
		var targetIndex = randi() % interestingObjects.size()
		var targetPosition = interestingObjects[targetIndex].to_global(Vector3())
		target = Vector2(targetPosition.x, targetPosition.z)
		targetID = interestingObjects[targetIndex].id
	else:
		target = Vector2()
		if position.x > 5:
			target.x = position.x - 5
		elif position.x < -5:
			target.x = position.x + 5
		if position.y > 5:
			target.y = position.y - 5
		elif position.y < -5:
			target.y = position.y + 5
```

The player's movement is controlled by this simple steering behaviour. It works by steering towards the current target and away from every object in the player's vision. Distance and size of objects determine how much to avoid them. Movement speed is determined based on curiosity. This code also rotates the player model correctly and plays the walking animation scaled to the current speed. If there is no target the player will stop and play the idle animation.

The code for avoiding obstacles is far from perfect and the player can sometimes be observed walking through trees. I wonder if I did myself a disservice by making the forest too dense for good obstacle avoidance, or perhaps I should have used a navmesh for movement.

```gd
if target and target.distance_squared_to(position) > 1:
	var toTarget = (target - position).normalized()
	var right = Vector2(-direction.y, direction.x)
	var wantedRotation = toTarget.dot(right)
	for index in range(obstacles.size()):
		var obstaclePosition = obstacles[index].to_global(Vector3())
		var toObstacle = (Vector2(obstaclePosition.x, obstaclePosition.z) - position).normalized()
		var scale = obstacles[index].get_parent().scale.x
		var weight = (scale - translation.distance_to(obstaclePosition)) / scale
		if weight < 0 or weight > 0.8:
			continue
		wantedRotation += (toObstacle.dot(right) - PI) * weight
	rotationY += wantedRotation * torque * delta
	direction.x = cos(rotationY)
	direction.y = sin(rotationY)
	translate(Vector3(direction.x, 0, direction.y) * speed * curiosity * delta)
	if target.distance_squared_to(Vector2(translation.x, translation.z)) <= 1:
		target = null
		targetID = 0
	$Astronaut.rotation = Vector3(0, PI / 2 - rotationY, 0)
	$Astronaut/AnimationPlayer.current_animation = "walk"
	$Astronaut/AnimationPlayer.playback_speed = animationSpeed * speed * curiosity
	footstepCountdown -= curiosity * delta
	if footstepCountdown <= 0.0:
		footstepCountdown = footstepInterval
		if randi() % 2 == 0:
			$FootAudio1.play()
		else:
			$FootAudio2.play()
else:
	target = null
	targetID = 0
	footstepCountdown = footstepStart
	$Astronaut/AnimationPlayer.current_animation = "idle"
	$Astronaut/AnimationPlayer.playback_speed = 1
```
