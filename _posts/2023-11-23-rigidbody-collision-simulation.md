---
date: 2022-11-23 01:35:41
layout: post
title: "Rigidbody collision simulation"
subtitle: 'Lab 1 of GAMES 103 Computer Animation'
description: >-
  This is part 1 of my course project at GAMES 103 Physics Simulation Series. 
  This part of the course work mainly focuses on simulating rigid body collision.
image: >-
  https://medias.wangruipeng.com/GAMES-103-Lab1-Res.png
optimized_image: >-
  https://medias.wangruipeng.com/GAMES-103-Lab1-Res.png
category: blog
tags:
  - physics
  - blog
author: Wang Ruipeng
paginate: true
---
# Rigid body Collision

This is part 1 of my course project at GAMES 103 Physics Simulation Series. This part of the course work mainly focuses on simulating rigid body collision.

Result:

[Click here to check the demo.](https://youtu.be/uGaLXmNSpgE)

First, let's take a look at all the possible variables used in the task, just to get familiar with them. This is to prevent getting dizzy and overwhelmed by the massive number of formulas later on.

```csharp
bool launched = false;
float dt  = 0.015f;
Vector3 v = new Vector3(0, 0, 0);	// velocity
Vector3 w = new Vector3(0, 0, 0);	// angular velocity
	
public float mass; // mass
Matrix4x4 I_ref; // reference inertia

float linear_decay = 0.999f;				// for velocity decay
float angular_decay = 0.98f;				
float restitution = 0.5f;                 // for collision
float friction = 0.2f;
```

Let's look at the first question, which requires us to update the object's position using Leapfrog integration in the update function. Additionally, if the launched variable is false, we will not perform the physical simulation.

> 1.a. Position update (2 Points) In the Update function, implement the update of the position
and the orientation by Leapfrog integration. Disable the linear motion and the position update, if launched is false.
(Hint: In general, try not to directly modify variables in transform, as doing this can occasionally slow down your simulation. Use temporary variables instead.)
> 

For this assignment, we need to use **leapfrog integration** method. Leapfrog integration is a numerical method widely used in computer physics simulations for its stability and efficiency, especially in dealing with differential equations in motion simulations. This method stands out for its ability to handle large time steps, making it ideal for real-time applications such as video games and visual effects in movies.

![Leapfrog](https://medias.wangruipeng.com/Leapfrog.png)

We just need to code according to the formula. First, we obtain `x0` and `q0` as the position and rotation angle of the object from the last simulation, and then update them after calculation.

```csharp
Vector3 x0 = transform.position;
Quaternion q0 = transform.rotation;

x = x0 + dt * v;
Vector3 dw = 0.5f * dt * w;
Quaternion qw = new Quaternion(dw.x, dw.y, dw.z, 0.0f);
q = Quaternion_Add(q0, qw * q0);

transform.position = x;
transform.rotation = q;
```

The second question requires us to update the object's velocity in the update function. The update of velocity needs to consider two influences: gravity and air friction. The problem specifies that the simulation of friction should be directly updated using the **linear_decay** and **angular_decay** coefficients provided in the script.

> 1.b. Velocity update (2 Points) Calculate the gravity force and use it to update the velocity.
To produce damping effects, you can multiply the velocities by linear and angular decay factors: v = c_linear decay v and ω = c_angular decay ω. Given the gravitation being the only force, you don’t need to calculate torque or to update the angular velocity. Now after you launch the bunny, you should see the bunny flying and spinning (with some initial angular velocity)!
> 

```csharp
v *= linear_decay;
w *= angular_decay;
```

Since the object is currently only subject to gravity, there is no need to update the angular velocity at this time. We only need to update the velocity v directly using the momentum formula.

```csharp
public Vector3 gravity = new Vector3(0.0f, -9.8f, 0.0f);
v += dt * gravity;
```

The third question requires us to perform collision detection inside the **`Collision_Impulse`** function and to calculate the average collision point.

> 1.c. Collision detection (2 Points) In your `Collision Impulse` function, calculate the position
and the velocity of every mesh vertex. Use them to determine whether the vertex is in collision with the floor. If so, add it to the sum and then compute the average of all colliding vertices.
> 

In the function, we determine whether the model vertices have collided with the plane. The function has two input parameters, which are three-dimensional vectors P and N. N refers to the normal vector of the plane, while P refers to any point on the plane. This allows us to define a plane. The task currently only requires us to perform collision detection with a plane, which is relatively simple and does not involve knowledge like Signed Distance Functions (SDF). It can be solved with high school mathematics, specifically solid geometry.

If we assume that the plane is infinitely large, for a point X on the model, we can calculate the angle between vector PX and the normal vector N. If the angle is less than 90 degrees, we can consider the point to be outside the plane; if it's greater than 90 degrees, the point can be considered inside the plane.
We use vector dot product to determine the angle between two vectors. Since we are only concerned with whether the angle is greater than 90 degrees, if the dot product is greater than 0, we can consider the point to be outside the plane, and vice versa.

There are some details to be aware of in the code implementation. First, the coordinates in **`vertices`** come from Unity's mesh component, so they are in the local coordinate system and need to be converted to the world coordinate system. Then, we can directly use the dot product for the calculation.

```csharp
Vector3 x_i = transform.TransformPoint(vertices[i]);
float d  = Vector3.Dot(x_i - P, N);
```

If `**d**` is less than 0, we can determine that the point is inside the plane. However, just determining that the point is inside the object is not enough. We also need to determine whether this point has any velocity moving towards the inside of the plane. If this point does not have any velocity towards the inside of the plane, it is not considered a collision point. The purpose of this is to prevent each point from being calculated as a collision point multiple times, which would result in excessive force applied.

According to rigid body dynamics, the velocity of each point is the vector sum of the overall velocity of the rigid body and the linear velocity caused by rotation. The linear velocity of rotation can be obtained by the cross product of the angular velocity and the distance from the rotation center to that point. Here, we assume that the rotation center is the origin of the object's local coordinate system, so it can be calculated. When writing code for rigid body simulation, it's crucial to always distinguish between 'the overall velocity of the object' and 'the velocity of a particular vertex on the object'. I encountered many pitfalls in this area.

```csharp
Matrix4x4 q_matrix = Matrix4x4.Rotate(q);
Vector3 Rri = q_matrix.MultiplyVector(vertices[i]);
Vector3 v_i = v + Vector3.Cross(w, Rri);
float v_N_Dot = Vector3.Dot(v_i, N);
```

Often, in each frame of the simulation, there will be multiple points located inside the plane. Therefore, we need to find all the points that are inside the plane and then calculate the position of a virtual average collision point. Here, we store the local coordinates of all the points we find in a List. Afterward, we calculate the position of the average collision point and store it in **`averageCollisionPoint`**

```csharp
List<Vector3> CollisionPoints = new List<Vector3>();
Matrix4x4 q_matrix = Matrix4x4.Rotate(q);

for (int i = 0; i < vertices.Length; i++)
{
   Vector3 x_i = transform.TransformPoint(vertices[i]);
   float d  = Vector3.Dot(x_i - P, N);
   if(d < 0.0f){
       Vector3 Rri = q_matrix.MultiplyVector(vertices[i]);
       Vector3 vi = v + Vector3.Cross(w, Rri);
       float viDotN = Vector3.Dot(vi, N);
       if(viDotN < 0.0f) CollisionPoints.Add(vertices[i]);
   }
}      
if (CollisionPoints.Count == 0) return;

Vector3 averageCollisionPoint = Vector3.zero;
for(int i = 0; i < CollisionPoints.Count; i++)
	averageCollisionPoint += CollisionPoints[i];
averageCollisionPoint /= CollisionPoints.Count;
Vector3 R_length = q_matrix.MultiplyVector(averageCollisionPoint);
Vector3 CollisionPointSpeed = v + Vector3.Cross(w, R_length);
```

After finding the position, we can then calculate the velocity of the point. The method to generate the point's velocity is still based on the previously mentioned approach: the cross product of the angular velocity and the distance from the rotation center to that point.

```csharp
Vector3 R_length = q_matrix.MultiplyVector(averageCollisionPoint);
Vector3 CollisionPointSpeed = v + Vector3.Cross(w, R_length);
```

The fourth question requires us to continue calculating the impulse j that the object receives after a collision.

> 1.d. Collision response (3 Points) In the same function, apply the impulse-based method to
calculate the proper impulse j for the average colliding position. You then update the linear velocity and the angular velocity by j accordingly. Doing this will enable the collision with the floor.
> 

The specific implementation is to directly follow the formula step by step. Here, we'll first present all the formulas. Don't worry, we will explain them one by one in detail.

![Rigid collision formulae](https://medias.wangruipeng.com/Rigid_collision_formulae.png)

We need to first calculate the velocity after the collision, the formula for which is as follows:

![Impulse](https://medias.wangruipeng.com/Impulse_method.png)

We decompose the velocity into components along the normal and tangential directions as **`CollisionPointSpeedN`** and **`CollisionPointSpeedF`**, respectively. Then, we calculate the velocity along the normal and tangential directions after the collision again, storing them in **`CollisionPointSpeedN_New`** and **`CollisionPointSpeedF_New`**, respectively. Finally, we combine these to form **`CollisionPointSpeed_New`**

```csharp
Vector3 CollisionPointSpeedN = N * Vector3.Dot(N, CollisionPointSpeed);
Vector3 CollisionPointSpeedF = CollisionPointSpeed - CollisionPointSpeedN;
Vector3 CollisionPointSpeedN_New = -restitution * CollisionPointSpeedN;
float a = Math.Max(1.0f - friction * (1.0f + restitution) * CollisionPointSpeedN.magnitude / CollisionPointSpeedF.magnitude, 0.0f);
Vector3 CollisionPointSpeedF_New = a * CollisionPointSpeedF;
Vector3 CollisionPointSpeed_New = CollisionPointSpeedN_New + CollisionPointSpeedF_New;
```

Then, based on the updated velocity, we calculate the generated impulse j, the formula for which is as follows:

![Response_by_impulse](https://medias.wangruipeng.com/Response_by_impulse.png)

![Response_by_impulse_2](https://medias.wangruipeng.com/Response_by_impulse_2.png)

Note that among all these formulas, only j is the unknown variable, so it's particularly easy to solve. Make sure to correspond the variables in the code with the ones in the formula above. Be aware that Unity's matrix operation library does not support scalar multiplication and subtraction with matrices, so we need to input these manually.

```csharp
Matrix4x4 RriStar = Get_Cross_Matrix(R_length);
Matrix4x4 IInverse = Matrix4x4.Inverse(q_matrix * I_ref * Matrix4x4.Transpose(q_matrix));
Matrix4x4 K = Matrix_Subtract(Matrix_Mulitiply(Matrix4x4.identity, 1.0f / mass), RriStar * IInverse * RriStar);
Vector3 J = K.inverse.MultiplyVector(CollisionPointSpeed_New - CollisionPointSpeed);
```

Finally, updating the velocity and rotation based on j will successfully complete the task!

```csharp
v += 1.0f / mass * J;
w += IInverse.MultiplyVector(Vector3.Cross(R_length, J));
```

Full code is attached below for reference. Do not copy it directly to your assignment!

```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using System;

public class Rigid_Bunny : MonoBehaviour 
{
	bool launched 		= false;
	float dt 			= 0.015f;
	Vector3 v 			= new Vector3(0, 0, 0);	// velocity
	Vector3 w 			= new Vector3(0, 0, 0);	// angular velocity
	
	public float mass;									// mass
	Matrix4x4 I_ref;							// reference inertia

	float linear_decay	= 0.999f;				// for velocity decay
	float angular_decay	= 0.98f;				
	float restitution 	= 0.5f;                 // for collision
    float friction = 0.2f;

    Mesh mesh;
    Vector3[] vertices;
	Vector3 x;
	Quaternion q;

    public Vector3 gravity = new Vector3(0.0f, -9.8f, 0.0f);

    // Use this for initialization
    void Start () 
	{
        mesh = GetComponent<MeshFilter>().mesh;
        vertices = mesh.vertices;

        float m=1;
		mass=0;
		for (int i=0; i<vertices.Length; i++) 
		{
			mass += m;
			float diag=m*vertices[i].sqrMagnitude;
			I_ref[0, 0]+=diag;
			I_ref[1, 1]+=diag;
			I_ref[2, 2]+=diag;
			I_ref[0, 0]-=m*vertices[i][0]*vertices[i][0];
			I_ref[0, 1]-=m*vertices[i][0]*vertices[i][1];
			I_ref[0, 2]-=m*vertices[i][0]*vertices[i][2];
			I_ref[1, 0]-=m*vertices[i][1]*vertices[i][0];
			I_ref[1, 1]-=m*vertices[i][1]*vertices[i][1];
			I_ref[1, 2]-=m*vertices[i][1]*vertices[i][2];
			I_ref[2, 0]-=m*vertices[i][2]*vertices[i][0];
			I_ref[2, 1]-=m*vertices[i][2]*vertices[i][1];
			I_ref[2, 2]-=m*vertices[i][2]*vertices[i][2];
		}
		I_ref [3, 3] = 1;
	}
	
	Matrix4x4 Get_Cross_Matrix(Vector3 a)
	{
		//Get the cross product matrix of vector a
		Matrix4x4 A = Matrix4x4.zero;
		A [0, 0] = 0; 
		A [0, 1] = -a [2]; 
		A [0, 2] = a [1]; 
		A [1, 0] = a [2]; 
		A [1, 1] = 0; 
		A [1, 2] = -a [0]; 
		A [2, 0] = -a [1]; 
		A [2, 1] = a [0]; 
		A [2, 2] = 0; 
		A [3, 3] = 1;
		return A;
	}

    private Matrix4x4 Matrix_Subtract(Matrix4x4 a, Matrix4x4 b)
    {
        for (int i = 0; i < 4; ++i)
            for (int j = 0; j < 4; ++j)            
                a[i, j] -= b[i, j];

        return a;
    }

    private Matrix4x4 Matrix_Mulitiply(Matrix4x4 a, float b)
    {
        for (int i = 0; i < 4; ++i)
            for (int j = 0; j < 4; ++j)
                a[i, j] *= b;
        return a;
    }

    private Quaternion Quaternion_Add(Quaternion a, Quaternion b)
    {
        a.x += b.x;
        a.y += b.y;
        a.z += b.z;
        a.w += b.w;
        return a;
    }

    // In this function, update v and w by the impulse due to the collision with
    //a plane <P, N>
    void Collision_Impulse(Vector3 P, Vector3 N)
	{
		List<Vector3> CollisionPoints = new List<Vector3>();
		Matrix4x4 q_matrix = Matrix4x4.Rotate(q);

        for (int i = 0; i < vertices.Length; i++)
		{
            Vector3 xi = transform.TransformPoint(vertices[i]);
			float d  = Vector3.Dot(xi - P, N);
			if(d < 0.0f)
			{
                Vector3 Rri = q_matrix.MultiplyVector(vertices[i]);
                Vector3 vi = v + Vector3.Cross(w, Rri);
                float viDotN = Vector3.Dot(vi, N);
				if(viDotN < 0.0f)
				{
					CollisionPoints.Add(vertices[i]);
				}
            }
        }

		if (CollisionPoints.Count == 0) return;

		Vector3 averageCollisionPoint = Vector3.zero;
		for(int i = 0; i < CollisionPoints.Count; i++)
		{
			averageCollisionPoint += CollisionPoints[i];
        }
		averageCollisionPoint /= CollisionPoints.Count;
        Vector3 R_length = q_matrix.MultiplyVector(averageCollisionPoint);
		Vector3 CollisionPointSpeed = v + Vector3.Cross(w, R_length);

		Vector3 CollisionPointSpeedN = N * Vector3.Dot(N, CollisionPointSpeed);
		Vector3 CollisionPointSpeedF = CollisionPointSpeed - CollisionPointSpeedN;
		Vector3 CollisionPointSpeedN_New = -restitution * CollisionPointSpeedN;
        float a = Math.Max(1.0f - friction * (1.0f + restitution) * CollisionPointSpeedN.magnitude / CollisionPointSpeedF.magnitude, 0.0f);
		Vector3 CollisionPointSpeedF_New = a * CollisionPointSpeedF;
		Vector3 CollisionPointSpeed_New = CollisionPointSpeedN_New + CollisionPointSpeedF_New;

        Matrix4x4 RriStar = Get_Cross_Matrix(R_length);
        Matrix4x4 IInverse = Matrix4x4.Inverse(q_matrix * I_ref * Matrix4x4.Transpose(q_matrix));
        Matrix4x4 K = Matrix_Subtract(Matrix_Mulitiply(Matrix4x4.identity, 1.0f / mass), RriStar * IInverse * RriStar);
        Vector3 J = K.inverse.MultiplyVector(CollisionPointSpeed_New - CollisionPointSpeed);

        v += 1.0f / mass * J;
        w += IInverse.MultiplyVector(Vector3.Cross(R_length, J));
    }

	// Update is called once per frame
	void Update () 
	{
		//Game Control
		if(Input.GetKey("r"))
		{
			transform.position = new Vector3 (0, 0.6f, 0);
			restitution = 0.5f;
			launched=false;
		}
		if(Input.GetKey("l"))
		{
			v = new Vector3 (5, 2, 0);
			launched=true;
		}

		if (launched)
		{
			// Part I: Update velocities
			v += dt * gravity;
			v *= linear_decay;
			w *= angular_decay;
			if (Vector3.Magnitude(v) <= 0.05f) launched = false;

			// Part II: Collision Impulse
			Collision_Impulse(new Vector3(0, 0.01f, 0), new Vector3(0, 1, 0));
			Collision_Impulse(new Vector3(2, 0, 0), new Vector3(-1, 0, 0));

			// Part III: Update position & orientation
			Vector3 x0 = transform.position;
			Quaternion q0 = transform.rotation;
			x = x0 + dt * v;
			Vector3 dw = 0.5f * dt * w;
			Quaternion qw = new Quaternion(dw.x, dw.y, dw.z, 0.0f);
			q = Quaternion_Add(q0, qw * q0);

			// Part IV: Assign to the object
			transform.position = x;
			transform.rotation = q;
		}
		
	}
}
```