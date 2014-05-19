---
layout: post
title: "DEFCON 2014 Quals : Zombies"
description: ""
category: writeups 
tags:
- defconquals2014
- 2014
- defconquals
- duchess
---
{% include JB/setup %}


### Description
    zombies
	Aim small, miss small. Or not...your call.
	zombies_8977dda19ee030d0ea35e97ad2439319.2014.shallweplayaga.me 20689

No binary? Either this is just a puzzle problem, or we're gonna have to pull a binary out of this...

Thankfully, its the former.

### Poking it with a stick

Its a physics game, where you have to shoot zombies on a 2-d plane. (distance to zombie, height of zombie above you).

You start out being given the option of choosing a rifle and a pistol to use, each with a different muzzle velocity (and unlisted magazine size!). It also tells you that the pistol may not work past 50m.

netcat time!

    Let's play a game...
	Situation: You're the last survivor of the zombie apocolypse in your town. You were driving your van of puppies to Tribbletown on a pleasant, windless day, and broke down in a valley. Now all 100 of your puppies are running amongst the zombies. Protect them with your guns!
	Choose your rifle. You will also have one spare magazine.
    Weapon                    Muzzle Velocity
    1. M1 Garand                 853
    2. LaRue OBR .556mm          975
    3. H&K G3 w/ box magazines   800
    2
    Choose your pistol. You will also have two spare magazines.
    Weapon                    Muzzle Velocity
    1. Wilson Combat CQB .45     251
    2. Desert Eagle .50          350
    3. Glock 17 9mm              375
    3
    Your weapons are: LaRue OBR .556mm and Glock 17 9mm
    Firing your pistol at a target beyond 50m is risky!
    Game mechanics: For each guess, enter 'r' or 'p' for rifle or pistol, followed by the angle of elevation for your shot and distance and elevation to the point you'd like your bullet to impact
    Example: r, 33.68, 12.44, 88.21
    You have your weapons, now watch your ammunition; Tribbletown will only let you in with all 100 puppies.

*  There are then 100 levels you have to hit a zombie on, with 0 misses.
*  Pick weapons with large (unlisted) magazine sizes, since we get a multiple of mag size for total ammo.
*  Command format is
    "{r,p}, angle, dest_distance, dest_height"



### Talking to it

We split the problem into two parts, one person writing a python script to play the game and parse levels, and one writing our firing calculations.
Initially, the levels only have a stationary zombie, and no time limit. Due to the way the levels were phrased. "You are standing still" and "The zombie is not moving" we assumed that eventually neither would be true. So rather than go with a simple closed form solution, we built a full simulator to guess shots.

While we did get this working, it was pretty slow, and had a few bugs with close-range high-up shots. Eventually, it got us to level 10, where things change a little.

*   It turns out you are not allowed to shoot past 50m with the pistols (we tried, in case we could ration ammo more carefully).
*   And if you try to switch what you shoot with in <1 second after level 10, you auto lose due to "transitions take time!".
*   There is no reloading
*   There is no reason to use anything but the highest ammo count weapons

### Levels
Levels look basically like

    --------------------------
    Level 1
	You are still, aiming carefully from your van
	The zombie is stalking a puppy 29m from your van and 35m above your van
	The zombie has no legs...it's not moving.
	Enter your shot details:
	
Starting with level 10, the zombies are moving, this we have (output from our tool):

    level 10
	Zombie @ 232.0:304.0 Moving to:116.0:304.0 in :11.0 seconds

Because calculating a moving shot is hard, and the game tells you how many seconds have passed

    1 second has passed...
	2 seconds have passed...

We just will always shoot after 2 seconds. So we aim at where the zombie will be in 2 seconds:

    target_time = ttd-1
    if ttd > 2:
        target_time = 2

    if(ttd > 0):
        tz = [0,0]

        dlta = target_time/(ttd)

        if(zd[0] > zm[0]):
            tz[0] = zd[0] - (zd[0]-zm[0])*dlta
        else:
            tz[0] = zd[0] + (zm[0]-zd[0])*dlta

        if(zd[1] > zm[1]):
            tz[1] = zd[1] - (zd[1]-zm[1])*dlta
        else:
            tz[1] = zd[1] + (zm[1]-zd[1])*dlta

Helpfully, it appears the zombies move in 1-second timesteps, which are slower than real-world seconds, so we can do this pretty easily.

At this point, unless we got a 'bad' point (and our shot compute got stuck), our script was able to beat levels pretty consistantly.

Our shot calculation work kept going, adding features to get ready to deal with a full 3-body problem while we continued running the game to see what would change.


Then our script hit level 91 without any changes occuring to the game and we realized maybe this wasn't going to get harder! However, since our shot calculations were so slow, and sometimes got stuck, we couldn't get far consistently.

### Closed form solution
Well, if its just going to be a single point not moving (since we decide when we will shoot ahead of time) we can just move to a closed form solution...

	alpha = -g/(2*v*v)
	a = alpha*x*x
	b = x
	c = -y+alpha*x*x
	D = b*b-4*a*c
	if d < 0: print "Can't hit!"
	psi = min((-b-sqrt(D))/(2*a), (-b+sqrt(D))/(2*a))
	theta = atan(psi)

There goes all our hard work!

As long as we make sure to always use the pistol when the target is < 50m away, we never run out of ammo, and this is able to hit almost 100% of the time.

(There was a very small inconsistency with our calculations vs the one on the server, and it would sometimes tell us that our angle would never hit the specified point, but since this was a ~1/200 chance, we didn't worry about it much)

### Solution
At this point, we just ran our script (with 15 minutes left in the competition!) and waited.

After 100 levels of hits, we get a flag.


	#!/usr/bin/python
	from sock import *
	import sys
	import time

	import struct

	import shot2

	host= "zombies_8977dda19ee030d0ea35e97ad2439319.2014.shallweplayaga.me:20689"

	rifles= [853,975,800]
	pistols = [251,350,375]

	rifle = 1
	pistol = 2


	riflefired = 0
	pistolfired = 0

	def go_interactive(con):
		while True:
			time.sleep(0.2)
			print con.read_one(0)
			con.write(sys.stdin.readline())


	def pause_script():
		raw_input("Paused... Press enter to continue")  

	def parse_opening(con,rifle,pistol):
		con.read_until("800")
		con.send_line(rifle)
		con.read_until("375")
		con.send_line(pistol)



	def parse_level(con):
		raw = con.read_until("Level ")
		lvl = int(con.read_until("\n"))

		# Parse what we are doing
		osstr = con.read_line()
		raw += osstr
		if "are still" in osstr:
			our_state = [0,0]
		else:
			print "WE ARE MOVING"
			print  osstr
			exit(1)

		# Parse zombie distance
		zdstr = con.read_line()
		raw += zdstr
		zdstr = zdstr.replace('m','')
		zombie_dist = [float(s) for s in zdstr.split(" ") if s.isdigit()]
		if len(zombie_dist) == 2:
			zmstr = con.read_line()
			raw += zmstr
			if "not moving" in zmstr:
				zombie_move = [0,0]
				time_to_die = [0]
			else:
				print "ZOMBIE IS MOVING"
				print zmstr
				exit(1)
		elif len(zombie_dist) == 4:
			zombie_move = [zombie_dist[2],zombie_dist[3]]
			timing = con.read_line()
			raw +=timing
			time_to_die = [float(s) for s in timing.split(" ") if s.isdigit()]
		else:
			print "Something bad in zd parsing"
			print zdstr



		#zombie movement parsing above


		check = con.read_line()
		raw +=check
		return(lvl,raw,our_state,zombie_dist,zombie_move,time_to_die[0],"shot details" in check)


	def print_level(lvl,os,zd,zm,ttd):
		print "Level "+str(lvl)
		print "Zombie @ "+str(zd[0])+":"+str(zd[1])+ " Moving to:"+str(zm[0])+":"+str(zm[1])+" in :"+str(ttd)
		print "We are moving:"+str(os[0])+":"+str(os[1])


	def my_calculate_shot(zd,vel):

		return (shot2.calculate_shot(vel,zd[0],zd[1]),zd[0],zd[1])

	def wait_for_time(con,tm):
		ct = 0
		while ct != tm:
			d = con.read_line()
			print d
			ct = [int(s) for s in d.split(" ") if s.isdigit()][0]


	def calculate_level(os,zd,zm,ttd):
		global pistolfired
		global riflefired


		# figure out which gun?
		if zd[0] <= 50:
			use_p = True
		else:
			use_p = False

		if use_p:
			(ang,dist,ele) = my_calculate_shot(zd,pistols[2])
			pistolfired +=1
			return "p, "+str(ang)+", "+str(zd[0])+", "+str(zd[1])
		else:
			(ang,dist,ele) = my_calculate_shot(zd,rifles[1])
			riflefired +=1
			return "r, "+str(ang)+", "+str(zd[0])+", "+str(zd[1])


	con = Sock(host)
	parse_opening(con,str(rifle+1),str(pistol+1))

	plvl = 0

	for i in range(0,100):
		(lvl,raw,os,zd,zm,ttd,ok) = parse_level(con)
		print raw

		if plvl > lvl:
			print "MISSED :("
			exit(1)
		else:
			plvl = lvl

		if not ok:
			print "Parse failure!"
			print raw
			exit(1)

		print_level(lvl,os,zd,zm,ttd)

		# Lets do the calculations for zombie moving in
		target_time = ttd-1
		if ttd > 2:
			target_time = 2

		if(ttd > 0):
			tz = [0,0]

			dlta = target_time/(ttd)

			if(zd[0] > zm[0]):
				tz[0] = zd[0] - (zd[0]-zm[0])*dlta
			else:
				tz[0] = zd[0] + (zm[0]-zd[0])*dlta

			if(zd[1] > zm[1]):
				tz[1] = zd[1] - (zd[1]-zm[1])*dlta
			else:
				tz[1] = zd[1] + (zm[1]-zd[1])*dlta

			# now we have the predicted target
		else:
			tz = zd

		move = calculate_level(os,tz,zm,ttd)
		print "Our move is:"+move

		# Wait until time to take the shot IF we have to
		if ttd > 0:
			wait_for_time(con,target_time)


		con.send_line(move)
		result = con.read_line()

		print "AMMO:R:"+str(riflefired)+"P:"+str(pistolfired)

		if "Sorry, you missed" in result:
			print "MISSED SHOT"
			print result
			exit(1)
		else:
			print result
	go_interactive(con)

And our shot calculation

    from math import *

	g = 9.81

	def calculate_shot(v,x,y):
		alpha = -g/(2*v*v)
		a = alpha*x*x
		b = x
		c = -y+alpha*x*x
		D = b*b-4*a*c
		if D < 0: return None
		psi = min((-b+sqrt(D))/(2*a), (-b+sqrt(D))/(2*a))
		theta = atan(psi)
		return degrees(theta)

Our (eventually un-used) shot calculation

	import math
	from collections import namedtuple

	Run = namedtuple('Run', ['angle', 'error', 'delta'])


	class Point:
		def __init__(self, x, y):
			self.x = x
			self.y = y
		def __repr__(self):
			return "P(%.4f, %.4f)"%(self.x, self.y)

		def distsq(self, point):
			return pow(self.x - point.x, 2) + pow(self.y - point.y, 2)

		def dist(self, point):
			return math.sqrt(pow(self.x - point.x, 2) + pow(self.y - point.y, 2))

		def mag(self,):
			return math.sqrt(pow(self.x, 2) + pow(self.y, 2))


	class PObject:
		def __init__(self, location, velocity, acceleration):
			self.location = location
			self.velocity = velocity
			self.acceleration = acceleration

		def update(self, timedelta):
			self.location.x += self.velocity.x * timedelta
			self.location.y += self.velocity.y * timedelta

			self.velocity.x += self.acceleration.x * timedelta
			self.velocity.y += self.acceleration.y * timedelta



	def simulate_shot( firing_solution, target_loc, timestep = 0.00001 ):
		speed,angle = firing_solution

		position = Point(0.,0.)
		velocity = Point(speed*math.cos(angle), speed*math.sin(angle))
		acceleration = Point(0, -9.81)

		shot = PObject( Point(0.,0.),
				Point(speed*math.cos(angle), speed*math.sin(angle)),
				Point(0, -9.81))

		time = 0
		while position.x < target_loc.x:
			lerp = 1

			if position.x + velocity.x * timestep > target_loc.x:
				# LERP
				lerp = (target_loc.x - position.x) / (velocity.x * timestep)

			position.x += lerp * velocity.x * timestep
			position.y += lerp * velocity.y * timestep

			velocity.x += lerp * acceleration.x * timestep
			velocity.y += lerp * acceleration.y * timestep

			time = time + lerp * timestep

			if time > 10:
				break
			if lerp != 1:
				break

		#print "POSITIONS", position, shot.location

		return time, position



	def find_shot( mps, target_loc, epsilon = 0.001, timestep = 0.00001 ):
		location = Point(0,0)
		solution = (mps, 0)

		time, position = simulate_shot( solution, target_loc, timestep )

		tries = []
		picks = set()
		limit = 500
		i = 1
		dontpanic = False

		#scaling = abs(math.cos(math.atan(target_loc.y / float(target_loc.x))) / (math.pi/2))
		#scaling = abs(target_loc.x) / target_loc.mag()

		while target_loc.dist(position) > epsilon:
			scaling = 1

			# print position,
			# print target_loc,
			# print solution,
			# print target_loc.dist(position),
			# print (target_loc.y - position.y),
			# print scaling,


			# print 0.4 * math.atan( (target_loc.y - position.y) / (location.x - position.x) ),

			# print

			# ensure the deltas have the same signs
			if False and not dontpanic and (len(tries) > 1 and tries[-1].error > tries[-2].error and
					tries[-1].delta * tries[-2].delta > 0):
				#if last_error is not None and current_error > last_error:
				# PANIC AND STUFF
				# don't use that generation to pick a new angle.
				# pick the best previous angle and mudge it a bit

				delta = tries[-1].delta + tries[-2].delta

				angledelta = tries[-1].angle + tries[-2].angle

				#winner = min(tries, key=lambda x: x.error)
				#angle = winner.angle + 0.001
				#while angle in picks:
				#    angle = angle + 0.01

				#picks.add(angle)
				dontpanic = True

				print "panicing, picking %.4f"%(angle)
				print winner
				print
			else:
				angle = solution[1] - 0.1 * math.atan( (target_loc.y - position.y) / (location.x - position.x) )
				dontpanic = False

			solution = (mps, angle)
			time, position = simulate_shot( solution, target_loc, timestep )
			tries.append( Run(angle, target_loc.dist(position), target_loc.y - position.y) )

			i += 1
			if i > limit:
				print "something has gone wrong"
				return None

		print "Landing shot at ", position
		print "Distance is ", target_loc.dist(position)
		print ( solution[1], position.x, position.y )

		return ( solution[1], position.x, position.y )


	def calculate_shot( mps, zombie_loc, puppy_loc, puppy_velocity, van_movement ):

		if puppy_loc is not None:
			raise "zombie movement not implemented"

		if puppy_velocity is not None:
			raise "puppy velocity not implemented"

		if van_movement is not None:
			raise "van movement not implemented"


		zombiep = Point(zombie_loc[0], zombie_loc[1])
		position = Point(0,0)

		angle, positionx, positiony = find_shot( mps, zombiep, timestep = (zombiep.dist(position)/mps)/50000 )
		return ( math.degrees(angle), positionx, positiony )
