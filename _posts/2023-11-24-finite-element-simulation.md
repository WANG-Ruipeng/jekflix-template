---
date: 2022-11-24 14:54:20
layout: post
title: "Finite Element Simulation"
subtitle: 'Lab 3 of GAMES 103 Computer Animation'
description: >-
  This is part 3 of my course project at GAMES 103 Physics Simulation Series. 
  This part of the course work mainly focuses on finite element simulation.
image: >-
  https://medias.wangruipeng.com/GAMES-103-Lab3-Res.png
optimized_image: >-
  https://medias.wangruipeng.com/GAMES-103-Lab3-Res.png
category: blog
tags:
  - physics
  - blog
author: Wang Ruipeng
paginate: false
---
# Finite Element Simulation

Result:

[Click here to check the demo.](https://youtu.be/2S4oOUNzfCY)
We open the project in Unity, and the scene should look like this.

![Lab3-1](https://medias.wangruipeng.com/Lab3-1.png)

Let's take a look at the code variables needed for this assignment, a slight impression will suffice.

```csharp
float dt = 0.003f;
float mass = 1;
float stiffness_0 = 20000.0f;
float stiffness_1 = 5000.0f;
float damp = 0.999f;

int[] Tet;
int tet_number;			//The number of tetrahedra

Vector3[] Force;
Vector3[] V;
Vector3[] X;
int number;				//The number of vertices

Matrix4x4[] inv_Dm;

//For Laplacian smoothing.
Vector3[] V_sum;
int[] V_num;

SVD svd = new SVD();
```

dt represents the time interval for each simulation. mass is the mass of the object. stiffness_0 and stiffness_1 correspond to s0 and s1, respectively. damp is the friction coefficient used in the simulation.

The Tet array is quite unique in this lab. Previously, our simulations' smallest unit was a triangle, but this time, it's a tetrahedron. Hence, the data in the Tet array is grouped in fours. For example, {0, 1, 2, 3, 0, 2, 3, 4} represents two tetrahedra, with the vertex indices of these tetrahedra being 0, 1, 2, 3, and 0, 2, 3, 4, respectively.

Force corresponds to the total force exerted on each point; V and X are the velocity and position of each point, respectively. If you have done labs 1 and 2, this should be familiar. inv_Dm will be used later. V_sum and V_num are for performing Laplacian smoothing later on. The assignment has already written the SVD decomposition for us, so there's no need to write it ourselves.

This is the first question:

> 1.a. Basic setup (2 Points) In the Update function, write the simulation of the house as a
simple particle system. Every vertex has its own position x and velocity v, and the velocity is under the influence of gravity. Please also implement frictional contact between every vertex and the floor. Since this project uses a relatively small time step, the Update function calls Update ten times.
After that, the function contains the code to send vertex positions into the house mesh for display.
> 

We find the code for applying gravity in the _Update function and add gravity to each point:

```csharp
for(int i=0 ;i<number; i++){
            //TODO: Add gravity to Force.
            Force[i] = new Vector3(0, -9.8f * mass, 0);
}
```

Next, we handle the velocity and position of each vertex. Similar to experiments one and two, but since we use the Force array to directly record the magnitude of the force, we need to divide it by the mass mass to calculate acceleration. Additionally, the velocity v also needs to be multiplied by the damp coefficient to simulate various types of energy loss:

```csharp
for (int i=0; i<number; i++){
            //TODO: Update X and V here.
            V[i] = (V[i] + dt * Force[i] / mass) * damp;
            X[i] = X[i] + dt * V[i];
}
```

The next step is to simulate collisions with the ground. Since the ground is a plane, we implement this directly using the impulse method. We simply compare the y-coordinate of each point; if it's less than the floor coordinate, we consider a collision to have occurred and give a corresponding velocity in the opposite direction. The magnitude of this velocity should be such that the point can return above the plane in the next simulation.

We just need to add collision detection in the above code segment, noting that FloorYPos is the floating-point variable that records the y-coordinate of the floor:

```csharp
for (int i=0; i<number; i++){
            //TODO: Update X and V here.
            V[i] = (V[i] + dt * Force[i] / mass) * damp;
            X[i] = X[i] + dt * V[i];

            //TODO: (Particle) collision with floor.
            if (X[i].y < FloorYPos)
            {
                V[i].y += (FloorYPos - X[i].y) / dt;
                X[i].y = FloorYPos;
            }
}
```

Now, let’s focus on question 2:

> 1.b. Edge matrices (2 Points). Next, write a Build Edge Matrix function that returns the edge
matrix of a tetrahedron. In the Start function, call this function to calculate inv Dm, the inverse of
the reference edge matrix for every tetrahedron
> 

The second question requires us to generate the corresponding edge matrices for all tetrahedra within the Build_Edge_Matrix function.

![Lab3-2](https://medias.wangruipeng.com/Lab3-2.png)

The slides in class were two-dimensional, but the code we need to write is three-dimensional; in fact, the difference is not significant. Note that since Unity only has 4x4 matrices, we actually need to expand the matrix. We can directly use the built-in functions in Unity, as follows:

```csharp
Matrix4x4 Build_Edge_Matrix(int tet){
    	Matrix4x4 ret=Matrix4x4.zero;
        //TODO: Need to build edge matrix here.
        Vector4 x10 = X[Tet[tet * 4 + 1]] - X[Tet[tet * 4]];
        Vector4 x20 = X[Tet[tet * 4 + 2]] - X[Tet[tet * 4]];
        Vector4 x30 = X[Tet[tet * 4 + 3]] - X[Tet[tet * 4]];
        Vector4 t = new Vector4(0, 0, 0, 1);
        ret.SetColumn (0, x10);
        ret.SetColumn (1, x20);
        ret.SetColumn (2, x30);
        ret.SetColumn (3, t);
        return ret;
}
```

Then we need to get `inv_Dm` for each tetrahedra within the `Start` function:

```csharp
//TODO: Need to allocate and assign inv_Dm
        inv_Dm = new Matrix4x4[tet_number];
        for (int tet = 0; tet < tet_number; tet++)
            inv_Dm[tet] = Build_Edge_Matrix(tet).inverse;
```

Then let’s turn to question 3:

![Lab3-3](https://medias.wangruipeng.com/Lab3-3.png)

Let’s review the formulae first:

![Lab3-4](https://medias.wangruipeng.com/Lab3-4.png)

We directly write the code for F, G, and S according to the formula, still paying attention to expanding the matrix from 3x3 to 4x4. Note that stiffness_0 and stiffness_1 correspond to s0 and s1, respectively.

```csharp
Matrix4x4 F = Build_Edge_Matrix(tet) * inv_Dm[tet];
//TODO: Green Strain
Matrix4x4 G = F.transpose * F;
G[0, 0] -= 1;
G[1, 1] -= 1;
G[2, 2] -= 1;
for (int i = 0; i < 3; i++)
   for (int j = 0; j < 3; j++)
        G[i, j] *= 0.5f;
//TODO: Second PK Stress
Matrix4x4 S = Matrix4x4.zero;
for (int i = 0; i < 3; i++)
    for (int j = 0; j < 3; j++)
         S[i, j] = 2 * stiffness_1 * G[i, j];
float A = stiffness_0 * (G[0, 0] + G[1, 1] + G[2, 2]);
S[0, 0] += A;
S[1, 1] += A;
S[2, 2] += A;
S[3, 3] = 1;
```

In the class, we discussed that P=FS, so we can directly write out the formula as well:

```csharp
//TODO: Elastic Force
Matrix4x4 P = F * S;
Matrix4x4 forces = P * inv_Dm[tet].transpose;
float B = -1 / (inv_Dm[tet].determinant * 6);
for (int i = 0; i < 3; i++)
    for (int j = 0; j < 3; j++)
        forces[i,j] *= B;
```

After we obtain the tensor matrix of the force, we can use the formula to update the velocities of the four corners of the tetrahedron, as follows:

```csharp
Force[Tet[tet * 4 + 0]] -= ((Vector3)forces.GetColumn(0) + (Vector3)forces.GetColumn(1) + (Vector3)forces.GetColumn(2));
Force[Tet[tet * 4 + 1]] += (Vector3)forces.GetColumn(0);
Force[Tet[tet * 4 + 2]] += (Vector3)forces.GetColumn(1);
Force[Tet[tet * 4 + 3]] += (Vector3)forces.GetColumn(2);
```

Then let's look at the fourth question. The task requires us to use Laplacian smoothing to make the simulation more realistic.

![Lab3-5](https://medias.wangruipeng.com/Lab3-5.png)

In machine learning, Laplacian smoothing is used to prevent a certain scenario from having a zero count. When compiling statistics, we add 1 to the sample count of each scenario. For example, for an event with three states A, B, and C, after 1000 observations, the number of occurrences for each state are 0, 990, and 10, respectively. Hence, the probabilities for K1 are 0, 0.99, and 0.01. To avoid giving event A a zero probability, we add 1 to the count of each scenario, resulting in 1, 991, and 11, for smoothing purposes. So, after Laplacian smoothing, the data becomes: 1/1003 = 0.001, 991/1003 = 0.988, 11/1003 = 0.011.

In our simulation here, to apply Laplacian smoothing, for each vertex, we need to calculate the average velocity of its adjacent vertices, and then update the original vertex velocity based on the average velocity of the adjacent vertices.

Since the code does not provide an adjacency matrix, we need to iterate according to the tetrahedra. The V_sum array stores the sum of the velocities of adjacent vertices for each vertex; the V_num array stores the count of adjacent vertices. The approach is similar to computing vertex normals based on face normals. For each tetrahedron, we calculate the velocity of all its vertices and add it to the V_sum array. This allows us to perform calculations bypassing the vertex adjacency matrix. The code is as follows:

```csharp
for (int i = 0; i < number; i++){
            V_sum[i] = new Vector3(0, 0, 0);
            V_num[i] = 0;
}

for (int tet = 0; tet < tet_number; tet++){
            Vector3 A = V[Tet[tet * 4 + 0]] + V[Tet[tet * 4 + 1]] + V[Tet[tet * 4 + 2]] + V[Tet[tet * 4 + 3]];
            V_sum[Tet[tet * 4 + 0]] += A;
            V_sum[Tet[tet * 4 + 1]] += A;
            V_sum[Tet[tet * 4 + 2]] += A;
            V_sum[Tet[tet * 4 + 3]] += A;
            V_num[Tet[tet * 4 + 0]] += 4;
            V_num[Tet[tet * 4 + 1]] += 4;
            V_num[Tet[tet * 4 + 2]] += 4;
            V_num[Tet[tet * 4 + 3]] += 4;
}
```

Once we obtain the average velocity, we can proceed with blending.

The specific formula may vary for each person, and the resulting effects may differ as well. When designing the formula, we need to pay attention to adhering as much as possible to the conservation of energy (although it certainly can't be fully conserved); in addition, we should note that our goal in designing the formula is to make the velocity smoother. When the average velocity of surrounding points is the same as the point itself, the velocity of the point itself should not change, since the velocity is already smooth. My personal code is as follows:

```csharp
for (int i = 0; i < number; i++){
            Vector3 AvgNeiVelocity = V_sum[i] / V_num[i];
            V[i] = (V[i] + 0.1f * AvgNeiVelocity) / 1.1f;
}
```

Then we are done now! If we run the code, we can see the shape being deformed: 

![Res](https://medias.wangruipeng.com/GAMES-103-Lab3-Res.png)

The full code is attached for reference:

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System;
using System.IO;

public class FVM : MonoBehaviour
{
	float dt 			= 0.003f;
    float mass 			= 1;
	float stiffness_0	= 20000.0f;
    float stiffness_1 	= 5000.0f;
    float damp			= 0.999f;

	int[] 		Tet;
	int tet_number;			//The number of tetrahedra

	Vector3[] 	Force;
	Vector3[] 	V;
	Vector3[] 	X;
	int number;				//The number of vertices

	Matrix4x4[] inv_Dm;

	//For Laplacian smoothing.
	Vector3[]   V_sum;
	int[]		V_num;

	SVD svd = new SVD();

    float FloorYPos;

    // Start is called before the first frame update
    void Start()
    {
    	// FILO IO: Read the house model from files.
    	// The model is from Jonathan Schewchuk's Stellar lib.
    	{
    		string fileContent = File.ReadAllText("Assets/house2.ele");
    		string[] Strings = fileContent.Split(new char[]{' ', '\t', '\r', '\n'}, StringSplitOptions.RemoveEmptyEntries);
    		
    		tet_number=int.Parse(Strings[0]);
        	Tet = new int[tet_number*4];

    		for(int tet=0; tet<tet_number; tet++)
    		{
				Tet[tet*4+0]=int.Parse(Strings[tet*5+4])-1;
				Tet[tet*4+1]=int.Parse(Strings[tet*5+5])-1;
				Tet[tet*4+2]=int.Parse(Strings[tet*5+6])-1;
				Tet[tet*4+3]=int.Parse(Strings[tet*5+7])-1;
			}
    	}
    	{
			string fileContent = File.ReadAllText("Assets/house2.node");
    		string[] Strings = fileContent.Split(new char[]{' ', '\t', '\r', '\n'}, StringSplitOptions.RemoveEmptyEntries);
    		number = int.Parse(Strings[0]);
    		X = new Vector3[number];
       		for(int i=0; i<number; i++)
       		{
       			X[i].x=float.Parse(Strings[i*5+5])*0.4f;
       			X[i].y=float.Parse(Strings[i*5+6])*0.4f;
       			X[i].z=float.Parse(Strings[i*5+7])*0.4f;
       		}
    		//Centralize the model.
	    	Vector3 center=Vector3.zero;
	    	for(int i=0; i<number; i++)		center+=X[i];
	    	center=center/number;
	    	for(int i=0; i<number; i++)
	    	{
	    		X[i]-=center;
	    		float temp=X[i].y;
	    		X[i].y=X[i].z;
	    		X[i].z=temp;
	    	}
		}
        /*tet_number=1;
        Tet = new int[tet_number*4];
        Tet[0]=0;
        Tet[1]=1;
        Tet[2]=2;
        Tet[3]=3;

        number=4;
        X = new Vector3[number];
        V = new Vector3[number];
        Force = new Vector3[number];
        X[0]= new Vector3(0, 0, 0);
        X[1]= new Vector3(1, 0, 0);
        X[2]= new Vector3(0, 1, 0);
        X[3]= new Vector3(0, 0, 1);*/

        //Create triangle mesh.
       	Vector3[] vertices = new Vector3[tet_number*12];
        int vertex_number=0;
        for(int tet=0; tet<tet_number; tet++)
        {
        	vertices[vertex_number++]=X[Tet[tet*4+0]];
        	vertices[vertex_number++]=X[Tet[tet*4+2]];
        	vertices[vertex_number++]=X[Tet[tet*4+1]];

        	vertices[vertex_number++]=X[Tet[tet*4+0]];
        	vertices[vertex_number++]=X[Tet[tet*4+3]];
        	vertices[vertex_number++]=X[Tet[tet*4+2]];

        	vertices[vertex_number++]=X[Tet[tet*4+0]];
        	vertices[vertex_number++]=X[Tet[tet*4+1]];
        	vertices[vertex_number++]=X[Tet[tet*4+3]];

        	vertices[vertex_number++]=X[Tet[tet*4+1]];
        	vertices[vertex_number++]=X[Tet[tet*4+2]];
        	vertices[vertex_number++]=X[Tet[tet*4+3]];
        }

        int[] triangles = new int[tet_number*12];
        for(int t=0; t<tet_number*4; t++)
        {
        	triangles[t*3+0]=t*3+0;
        	triangles[t*3+1]=t*3+1;
        	triangles[t*3+2]=t*3+2;
        }
        Mesh mesh = GetComponent<MeshFilter> ().mesh;
		mesh.vertices  = vertices;
		mesh.triangles = triangles;
		mesh.RecalculateNormals ();

		V 	  = new Vector3[number];
        Force = new Vector3[number];
        V_sum = new Vector3[number];
        V_num = new int[number];

        GameObject floorObj = GameObject.Find("Floor");
        FloorYPos = floorObj.transform.position.y;

        //TODO: Need to allocate and assign inv_Dm
        inv_Dm = new Matrix4x4[tet_number];
        for (int tet = 0; tet < tet_number; tet++)
            inv_Dm[tet] = Build_Edge_Matrix(tet).inverse;
    }

    Matrix4x4 Build_Edge_Matrix(int tet)
    {
    	Matrix4x4 ret=Matrix4x4.zero;
        //TODO: Need to build edge matrix here.
        Vector4 x10 = X[Tet[tet * 4 + 1]] - X[Tet[tet * 4]];
        Vector4 x20 = X[Tet[tet * 4 + 2]] - X[Tet[tet * 4]];
        Vector4 x30 = X[Tet[tet * 4 + 3]] - X[Tet[tet * 4]];
        Vector4 t = new Vector4(0, 0, 0, 1);
        ret.SetColumn (0, x10);
        ret.SetColumn (1, x20);
        ret.SetColumn (2, x30);
        ret.SetColumn (3, t);
        return ret;
    }

    void _Update()
    {
    	// Jump up.
		if(Input.GetKeyDown(KeyCode.Space))
    	{
    		for(int i=0; i<number; i++)
    			V[i].y+=0.2f;
    	}

    	for(int i=0 ;i<number; i++)
    	{
            //TODO: Add gravity to Force.
            Force[i] = new Vector3(0, -9.8f * mass, 0);
        }

    	for(int tet=0; tet<tet_number; tet++)
    	{
            //TODO: Deformation Gradient
            Matrix4x4 F = Build_Edge_Matrix(tet) * inv_Dm[tet];
            //TODO: Green Strain
            Matrix4x4 G = F.transpose * F;
            G[0, 0] -= 1;
            G[1, 1] -= 1;
            G[2, 2] -= 1;
            for (int i = 0; i < 3; i++)
                for (int j = 0; j < 3; j++)
                    G[i, j] *= 0.5f;
            //TODO: Second PK Stress
            Matrix4x4 S = Matrix4x4.zero;
            for (int i = 0; i < 3; i++)
                for (int j = 0; j < 3; j++)
                    S[i, j] = 2 * stiffness_1 * G[i, j];
            float A = stiffness_0 * (G[0, 0] + G[1, 1] + G[2, 2]);
            S[0, 0] += A;
            S[1, 1] += A;
            S[2, 2] += A;
            S[3, 3] = 1;
            //TODO: Elastic Force
            Matrix4x4 P = F * S;
            Matrix4x4 forces = P * inv_Dm[tet].transpose;
            float B = -1 / (inv_Dm[tet].determinant * 6);
            for (int i = 0; i < 3; i++)
                for (int j = 0; j < 3; j++)
                    forces[i,j] *= B;

            /*Force[Tet[tet * 4 + 0]] -= ((Vector3)forces.GetColumn(0) + (Vector3)forces.GetColumn(1) + (Vector3)forces.GetColumn(2));
            Force[Tet[tet * 4 + 1]] += (Vector3)forces.GetColumn(0);
            Force[Tet[tet * 4 + 2]] += (Vector3)forces.GetColumn(1);
            Force[Tet[tet * 4 + 3]] += (Vector3)forces.GetColumn(2);
            */

            Force[Tet[tet * 4 + 0]].x -= (forces[0, 0] + forces[0, 1] + forces[0, 2]);
            Force[Tet[tet * 4 + 0]].y -= (forces[1, 0] + forces[1, 1] + forces[1, 2]);
            Force[Tet[tet * 4 + 0]].z -= (forces[2, 0] + forces[2, 1] + forces[2, 2]);
            Force[Tet[tet * 4 + 1]].x +=  forces[0, 0];
            Force[Tet[tet * 4 + 1]].y +=  forces[1, 0];
            Force[Tet[tet * 4 + 1]].z +=  forces[2, 0];
            Force[Tet[tet * 4 + 2]].x +=  forces[0, 1];
            Force[Tet[tet * 4 + 2]].y +=  forces[1, 1];
            Force[Tet[tet * 4 + 2]].z +=  forces[2, 1];
            Force[Tet[tet * 4 + 3]].x +=  forces[0, 2];
            Force[Tet[tet * 4 + 3]].y +=  forces[1, 2];
            Force[Tet[tet * 4 + 3]].z +=  forces[2, 2];

        }

        for (int i = 0; i < number; i++)
        {
            V_sum[i] = new Vector3(0, 0, 0);
            V_num[i] = 0;
        }

        for (int tet = 0; tet < tet_number; tet++)
        {
            Vector3 A = V[Tet[tet * 4 + 0]] + V[Tet[tet * 4 + 1]] + V[Tet[tet * 4 + 2]] + V[Tet[tet * 4 + 3]];
            V_sum[Tet[tet * 4 + 0]] += A;
            V_sum[Tet[tet * 4 + 1]] += A;
            V_sum[Tet[tet * 4 + 2]] += A;
            V_sum[Tet[tet * 4 + 3]] += A;
            V_num[Tet[tet * 4 + 0]] += 4;
            V_num[Tet[tet * 4 + 1]] += 4;
            V_num[Tet[tet * 4 + 2]] += 4;
            V_num[Tet[tet * 4 + 3]] += 4;
        }

        for (int i = 0; i < number; i++)
        {
            Vector3 AvgNeiVelocity = V_sum[i] / V_num[i];
            V[i] = (V[i] + 0.1f * AvgNeiVelocity) / 1.1f;
        }

        for (int i=0; i<number; i++)
    	{
            //TODO: Update X and V here.
            V[i] = (V[i] + dt * Force[i] / mass) * damp;
            X[i] = X[i] + dt * V[i];

            //TODO: (Particle) collision with floor.
            if (X[i].y < FloorYPos)
            {
                V[i].y += (FloorYPos - X[i].y) / dt;
                X[i].y = FloorYPos;
            }
        }
    }

    // Update is called once per frame
    void Update()
    {
    	for(int l=0; l<10; l++)
    		 _Update();

    	// Dump the vertex array for rendering.
    	Vector3[] vertices = new Vector3[tet_number*12];
        int vertex_number=0;
        for(int tet=0; tet<tet_number; tet++)
        {
        	vertices[vertex_number++]=X[Tet[tet*4+0]];
        	vertices[vertex_number++]=X[Tet[tet*4+2]];
        	vertices[vertex_number++]=X[Tet[tet*4+1]];
        	vertices[vertex_number++]=X[Tet[tet*4+0]];
        	vertices[vertex_number++]=X[Tet[tet*4+3]];
        	vertices[vertex_number++]=X[Tet[tet*4+2]];
        	vertices[vertex_number++]=X[Tet[tet*4+0]];
        	vertices[vertex_number++]=X[Tet[tet*4+1]];
        	vertices[vertex_number++]=X[Tet[tet*4+3]];
        	vertices[vertex_number++]=X[Tet[tet*4+1]];
        	vertices[vertex_number++]=X[Tet[tet*4+2]];
        	vertices[vertex_number++]=X[Tet[tet*4+3]];
        }
        Mesh mesh = GetComponent<MeshFilter> ().mesh;
		mesh.vertices  = vertices;
		mesh.RecalculateNormals ();
    }
}
```