---
date: 2022-11-25 03:19:52
layout: post
title: "Water Ripple"
subtitle: 'Lab 4 of GAMES 103 Computer Animation'
description: >-
  This is part 4 of my course project at GAMES 103 Physics Simulation Series. 
  This part of the course work mainly focuses on water ripple simulation: mainly focus on shallow water.
image: >-
  https://medias.wangruipeng.com/GAMES-103-Lab4-Res.png
optimized_image: >-
  https://medias.wangruipeng.com/GAMES-103-Lab4-Res.png
category: blog
tags:
  - physics
  - blog
author: Wang Ruipeng
paginate: false
---
# Pool Ripple

Let’s take a look a first problem:

> 1.a. Basic setup (1 Point) At the beginning of the update function, load the heights (the y values) of the vertices into h. At the end of the update function, set h back into the heights of the vertices. Remember to recalculate the mesh normal in the end.
> 

The first question is very simple. Since our data is operated within the array h, we need to synchronize the data at the beginning and end of each frame, syncing h to the mesh data. We just need to synchronize at the corresponding positions. By reading the code in the start function that generates the mesh, we find that the vertices of the water surface are arranged in rows and columns, and it's a one-dimensional array. Therefore, we also need to be mindful of this during synchronization. The code is as follows. Synchronize at the beginning of the update function:

```csharp
//TODO: Load X.y into h.
        for (int i = 0; i < size; i++)
            for (int j = 0; j < size; j++)
                h[i, j] = X[i * size + j].y;
```

We also need to perform corresponding synchronization at the end of the update function, syncing h to the mesh:

```csharp
//TODO: Store h back into X.y and recalculate normal.
        for (int i = 0; i < size; i++)
            for (int j = 0; j < size; j++)
                X[i * size + j].y = h[i, j];
        mesh.vertices = X;
```

Now, let’s look at question 2:

> 1.b. User Interaction (1 Point) When the player hits the ‘r’ key, add a random water drop into
the pool. To do so, you can increase the column h[i,j] by r, in which i, j and r are all randomly
determined. One problem is that it can cause more water volume to be poured into the pool over time, according to the shallow wave model. To solve this problem, simply deduce the same amount (r) from the surrounding columns so the total volume stays the same.
After implementing the above procedure, you should observe a bump on the surface whenever the key gets pressed, but no wave propagation yet.
> 

We need to implement the propagation of water droplets. We only need to increase the height by a random amount r in a random cell (i, j). Of course, to prevent the total volume of water in the pool from increasing, we need to reduce the same amount of water in the surrounding cells to keep the total volume of water in the pool constant. The code is quite simple:

```csharp
if (Input.GetKeyDown ("r")) {
	//TODO: Add random water.
	    int i = Random.Range(1, size - 1);
            int j = Random.Range(1, size - 1);

            float r = Random.Range(0.5f, 2.0f);
            h[i, j] += r;
            h[i - 1, j] -= r / 8;
            h[i - 1, j - 1] -= r / 8;
            h[i + 1, j] -= r / 8;
            h[i + 1, j + 1] -= r / 8;
            h[i, j - 1] -= r / 8;
            h[i + 1, j - 1] -= r / 8;
            h[i, j + 1] -= r / 8;
            h[i - 1, j + 1] -= r / 8;
        }
```

Now the effect we have obtained is as follows: the water level in the pool has changed, but there is currently no propagation on the water surface, so it looks somewhat like stalactites.

![Lab4-1](https://medias.wangruipeng.com/Lab4-1.png)

Now we need to take a look at the next problem:

![Lab4-2](https://medias.wangruipeng.com/Lab4-2.png)

Now it's time to update the heights according to the formula. The formula presented in the class actually tells us how to implement this. It's particularly important to pay attention to the boundary conditions, that is, the simulation of the water flow grid at the edges, which needs to be modeled using Neumann conditions. It sounds quite sophisticated, but in reality, it simply assumes that there is no exchange of water between the water flow and the walls. Therefore, in the formula, the adjacent parts also need to remove the corresponding non-existent parts. The code is as follows:

```csharp
for (int i = 0; i < size; i++)
		{
			for (int j = 0; j < size; j++)
			{
				new_h[i, j] = h[i, j] + (h[i, j] - old_h[i, j]) * damping;

				if (i != 0) new_h[i, j] += (h[i - 1, j] - h[i, j]) * rate;
				if (i != size - 1) new_h[i, j] += (h[i + 1, j] - h[i, j]) * rate;
				if (j != 0) new_h[i, j] += (h[i, j - 1] - h[i, j]) * rate;
				if (j != size - 1) new_h[i, j] += (h[i, j + 1] - h[i, j]) * rate;
			}
		}
```

Then comes the update of the arrays old_h, h, and new_h, which is also very straightforward:

```csharp
//Step 3
        //TODO: old_h <- h; h <- new_h;
        for (int i = 0; i < size; i++)
		{
			for (int j = 0; j < size; j++)
			{
                old_h[i, j] = h[i, j];
                h[i, j] = new_h[i, j];
            }
			
		}
```

This is the result now:

![Lab4-3](https://medias.wangruipeng.com/Lab4-3.png)

Now, let’s look at the last step:

![Lab4-4](https://medias.wangruipeng.com/Lab4-4.png)

We need to first find all the grids that intersect with the cube and then mark them, so we need to get the coordinates of the cube first.

```csharp
GameObject Cube = GameObject.Find("Block");
Vector3 cube_p = Cube.transform.position;
Mesh cube_mesh = Cube.GetComponent<MeshFilter>().mesh;
```

The plane in the project is 10x10 in size, with the center of the plane at the origin (0,0), and the water surface mesh is composed of 100x100 small squares. Therefore, to convert the cube's coordinates to the water surface's coordinates, a linear transformation is also needed (I didn't understand how to do this part until I saw the example given by the teacher, and then I immediately understood). Finally, we can calculate the range of the water surface mesh covered by the cube, stored in upper_i, upper_j, lower_i, and lower_j:

```csharp
int lower_i = (int)((cube_p.x + 5.0f) * 10) - 3;
int upper_i = (int)((cube_p.x + 5.0f) * 10) + 3;
int lower_j = (int)((cube_p.z + 5.0f) * 10) - 3;
int upper_j = (int)((cube_p.z + 5.0f) * 10) + 3;
Bounds bounds = cube_mesh.bounds;
```

After that, calculate the required grid height, stored in the array low_h. We can directly use Unity's built-in AABB bounding box and ray to calculate. Still need to pay attention to the transformation between local coordinates and world coordinates:

```csharp
for (int i = lower_i - 3; i <= upper_i + 3; i++){
    for (int j = lower_j - 3; j <= upper_j + 3; j++){
        if (i >= 0 && j >= 0 && i < size && j < size){
            Vector3 p = new Vector3(i * 0.1f - size * 0.05f, -11, j * 0.1f - size * 0.05f);
            Vector3 q = new Vector3(i * 0.1f - size * 0.05f, -10, j * 0.1f - size * 0.05f);
            p = Cube.transform.InverseTransformPoint(p);
            q = Cube.transform.InverseTransformPoint(q);

            Ray ray = new Ray(p, q - p);
            float dist = 99999;
            bounds.IntersectRay(ray, out dist);

            low_h[i, j] = -11 + dist;//cube_p.y-0.5f;
        }
    }
}
```

We can use low_h and the original h for judgment, and then we can determine which heights need to be updated, and store them inside the cg_mask matrix. We can also calculate the matrix b needed in the formula:

```csharp
for (int i = 0; i < size; i++){
    for (int j = 0; j < size; j++)
    {
        if (low_h[i, j] > h[i, j])
        {
            b[i, j] = 0;
            vh[i, j] = 0;
            cg_mask[i, j] = false;
        }
        else
        {
            cg_mask[i, j] = true;
            b[i, j] = (new_h[i, j] - low_h[i, j]) / rate;
        }
    }
}
```

Finally, it's time to call the function already written in the problem to solve the Poisson equation:

```csharp
Conjugate_Gradient(cg_mask, b, vh, lower_i - 1, upper_i + 1, lower_j - 1, upper_j + 1);
```

Take a look at the result:

![Result](https://medias.wangruipeng.com/GAMES-103-Lab4-Res.png)