+++
date = "2012-07-19T12:52:03+01:00"
draft = false
title = "Quaternions and Dual Quaternion Skinning"
tags = [ "quaternions", "animation" ]
+++


For some reason I like quaternions. I fell in love with complex numbers back in school when I found out that they [made more sense than real numbers](http://en.wikipedia.org/wiki/Algebraically_closed_field#Examples). While it [might not exactly be helpful](http://www.geometricalgebra.net/quaternions.html) to visualise quaternions as an extension of complex numbers, there's something in there that just grabs at me. Unlike previous posts, I've managed to update to D3D11 so I'll be discussing implementation details in terms of HLSL (Shader Model 4, as I also have a D3D10 dev machine).

I'm no mathematician so hopefully this information in this post should be pretty accessible.


##### Dual Quaternion Skinning


I spent a couple of hours last week converting my skinning pipeline to use dual quaternions. My animation pipeline works with quaternions; the source animation data, skeleton pose and inverse base pose are quaternion-based. Right at the very end, the composited result is converted to matrix and sent to the GPU. In effect, I'm doing this:

~~~cpp
for (int i = 0; i < nb_bones; i++)
{
	const rend::Bone& bone = skeleton.bones[i];
	const rend::Keyframe& kf = keyframes[i + frame * nb_bones];

	// Calculate inverse base pose
	math::quat q0 = qConjugate(bone.base_pose.rotation);
	math::vec3 p0 = qTransformPos(q0, v3Negate(bone.base_pose.position));

	// Concatenate with animation keyframe
	math::quat q = qMultiply(q0, kf.rotation);
	math::vec3 p = qTransformPos(kf.rotation, p0);
	p = v3Add(p, kf.position);

	// Set the transform
	math::mat4& transform = transforms[i];
	transform = qToMat4(q);
	transform.r[3] = math::v4Make(p.x, p.y, p.z, 1);
}
~~~

In reality, I precalculate the inverse base pose, keeping the inner loop tight and low on ALU operations. My goals were:

* Capitalise on the quaternion input and remove the conversion to matrix step. On all past games I've worked on, the matrix multiply with base pose has had a destructive influence on CPU performance; if I could remove this step, I'd have a solution that would be faster than my past implementations.
* Reduce the amount of data being mapped/sent to the GPU. While my existing solution was sending 4x4 matrices, one of the columns was redundant - I could be sending 4x3. However, dual quaternions would allow me to halve the amount of data sent.
* Get volume-preserving skinning on joints under extreme rotation.
* Have a bit of fun and exercise some neglected quaternion/vector math muscles.

 I skimmed the paper, copied the HLSL source code and tested everything on a basic idle animation. Bad idea, right? But everything seemed to work. I hooked up my mocap pipeline this week and everything went wrong - limbs were folding inside each other and the character was skewing all over the place.

 Searching the internets for equivalent implementations found that most basically copied and pasted from the paper. Some seemed to make an effort to understand what was going on under the hood but all of them produced the same results (oddly, the nVidia shader was the most needlessly complicated and inefficient of them all). So I knuckled down and decided to read the following papers in more detail:

 * [A Beginners Guide to Dual Quaternions](http://wscg.zcu.cz/wscg2012/cd-rom/short/A29-full.pdf)
 * [Geometric Skinning with Approximate Dual Quaternion Blending](http://isg.cs.tcd.ie/projects/DualQuaternions/)

The basic method is this:

* Convert your quaternion rotation and position vector to a dual quaternion on the CPU for each bone.
* In your vertex shader, load all dual quaternion bone transforms for the vertex.
* Create a single dual quaternion that is the weighted average of all the bone transforms.
* Transform the vertex position and normal by this dual quaternion.


##### Quaternion/Vector Multiplication


To cut a long story short, the reason none of this was working for me was that I was looking at Cg code, as opposed to HLSL code. Having never really used Cg, it never occurred to me that the order of cross product parameters should be swapped to account for handedness. Cross products are used to define the quaternion multiplication:

$$ Q = (w, V) \\\ Q_r = Q_0 Q_1  \\\ Q_r = (w_0 w_1 - V_0 \cdot V_1, w_0 V_1 + w_1 V_0 + V_0 \times V_1)$$

<center> ($\cdot$ is the vector dot product and $\times$ is the vector cross product) </center>

Naturally, this is then a non-commutative operation, giving the order in which you pass arguments into your multiply functions importance, too. A basic C++ implementation of the multiplication looks like:

~~~cpp
math::quat math::qMultiply(const math::quat& a, const math::quat& b)
{
	quat q =
	{
		a.w * b.x + a.x * b.w + a.y * b.z - a.z * b.y,
		a.w * b.y - a.x * b.z + a.y * b.w + a.z * b.x,
		a.w * b.z + a.x * b.y - a.y * b.x + a.z * b.w,
		a.w * b.w - a.x * b.x - a.y * b.y - a.z * b.z,
	};
	return q;
}
~~~

Assuming we're using unit quaternions, you can rotate a vector by a quaternion using multiplication:

$$P_r = Q*(0, P)*Q'$$

Here, $*$ is the quaternion multiplication, $'$ is the quaternion conjugate and $(0,P)$ is construction of a pure quaternion using the vector, setting the quaternion real component to zero. This equation is used to convert to and from dual quaternions, so it's important you keep order of multiplications in mind.

If you work with Cg, make sure you have your cross products round the other way to my shader examples below!


##### Converting to Dual Quaternion


I'd suggest reading the second paper linked above to get a good introduction to what dual quaternions are and what the various components mean. For our means here it's enough to say that dual quaternions are just a pair of quaternions - the "real" quaternion and the "dual" quaternion:

$$DQ = (Q_r, Q_d)$$

Converting a quaternion rotation and position vector to this representation is simple. The $Q_r$ term is a simple copy of your quaternion rotation and $Q_d$ is:

$$Q_d = 0.5(0, V)*Q_r$$

This allows the matrix conversion code on the CPU to become:

~~~cpp
// Convert position/rotation to dual quaternion
math::dualquat& transform = transforms[i];
transform.r = q;
transform.d = qScale(qMultiplyPure(q, p), 0.5f);
~~~

Much nicer! `MultiplyPure` is a simple function that does a special case multiply where `p.w` is zero.


##### Blending Dual Quaternions


Now we can move over to the HLSL side. This is my transform loading code:

~~~cpp
Buffer<float4> g_BoneTransforms;

float2x4 GetBoneDualQuat(uint bone_index)
{
	// Load bone rows individually
	float4 row0 = g_BoneTransforms.Load(bone_index * 2 + 0);
	float4 row1 = g_BoneTransforms.Load(bone_index * 2 + 1);
	return float2x4(row0, row1);
}
~~~

As mentioned before, we want to load the bones that influence a vertex, weight them and transform the vertex position and normal:

~~~cpp
float2x4 BlendBoneTransforms(uint4 bone_indices, float4 bone_weights)
{
	// Fetch bones
	float2x4 dq0 = GetBoneDualQuat(bone_indices.x);
	float2x4 dq1 = GetBoneDualQuat(bone_indices.y);
	float2x4 dq2 = GetBoneDualQuat(bone_indices.z);
	float2x4 dq3 = GetBoneDualQuat(bone_indices.w);

	// ...blend...
}
~~~

As with quaternions, weighting dual quaternions can be achieved using a normalised lerp of its components. As explained in the paper [Understanding Slerp, Then Not Using It](http://number-none.com/product/Understanding%20Slerp,%20Then%20Not%20Using%20It/), it's not as good as a SLERP in that it's not constant velocity. However, it has minimal torque, rotating along the sphere and unlike SLERP, is commutative; meaning, if you combine multiple quaternions in a different order, the result will always be the same.

Besides, when you're regenerating bone rotations each frame, interpolation velocity won't really factor into the solution. The weighting code thus becomes:

~~~cpp
// Blend
float2x4 result =
	bone_weights.x * dq0 +
	bone_weights.y * dq1 +
	bone_weights.z * dq2 +
	bone_weights.w * dq3;

// Normalise
float norm = length(result[0]);
return result / norm;
~~~

Simples! In the original paper, Kavan goes into great detail on why this works so well over previous solutions and why a SLERP isn't the ideal solution. Well worth a read.


##### Antipodality or, Quaternion Double-Cover


Take a look at one of the classic release bugs of this generation:

<center> <iframe width="420" height="315" src="http://www.youtube.com/embed/ToKIkw3LIoQ" frameborder="0" allowfullscreen></iframe></center>

There's a pretty awesome reason for why the head can spin the long way around to get to its target rotation (no doubt the bug will be more complicated than that). Casey Muratori goes into [great detail on this](https://mollyrocket.com/837) and I'll attempt to summarise.

The axis-angle definition of a quaternion is:

$$Q = (cos(\frac{\theta}{2}), Vsin(\frac{\theta}{2}))$$

Inherent in this definition is the ability for quaternions to represent up to 720 degrees of rotation. Assuming we use the x-axis as our example vector, this leads to these quantities:

$$Q(0) = (1, 0, 0, 0) \\\ Q(360) = (-1, 0, 0, 0) \\\ Q(720) = (1, 0, 0, 0)$$

While sine and cosine are periodic every 360 degrees, the division by two of the input angle leads the quaternion representation to be periodic every 720 degrees.

Clearly, 360 degrees and 720 degrees represent the same geometrical rotation. However, when you interpolate between rotations, you may find yourself interpolating the long way round. Your source/target rotation range may geometrically be only 30 to 35 degrees but your quaternion may represent that as 30 to 395 degrees!

If you've glimpsed at the inner workings of a SLERP, this is the case they are trying to avoid when trying to solve for the "shortest path". Given that $Q$ and $-Q$ represent the same rotation, you can ensure interpolation between two quaternions follows this shortest path by negating one of the quaternions if the dot product between them is negative.

When blending dual quaternions, you have to watch for the same case. Not only that, you have to ensure that all of your bone transforms are in the same neighbourhood. While there are [complicated ways of achieving that](http://pages.cs.wisc.edu/~joony/RESEARCH/online_locomotion.pdf), most of the time it can be guaranteed by comparing all bone rotations to the first one and adjusting the sign of the blend weight.

This leads to the final code:

~~~cpp
float2x4 BlendBoneTransforms(uint4 bone_indices, float4 bone_weights)
{
	// Fetch bones
	float2x4 dq0 = GetBoneDualQuat(bone_indices.x);
	float2x4 dq1 = GetBoneDualQuat(bone_indices.y);
	float2x4 dq2 = GetBoneDualQuat(bone_indices.z);
	float2x4 dq3 = GetBoneDualQuat(bone_indices.w);

	// Ensure all bone transforms are in the same neighbourhood
	if (dot(dq0[0], dq1[0]) < 0.0) bone_weights.y *= -1.0;
	if (dot(dq0[0], dq2[0]) < 0.0) bone_weights.z *= -1.0;
	if (dot(dq0[0], dq3[0]) < 0.0) bone_weights.w *= -1.0;

	// Blend
	float2x4 result =
		bone_weights.x * dq0 +
		bone_weights.y * dq1 +
		bone_weights.z * dq2 +
		bone_weights.w * dq3;

	// Normalise
	float norm = length(result[0]);
	return result / norm;
}
~~~

There are cases which can still fail these checks but they are very rare for the general use-case of skinning - I can't imagine this working well for severe joint twists, for example (beyond the range of human constraints, that is).


##### Transforming the Vertex with Dual Quaternions


Once you have the blended result you need to convert it back into quaternion/vector form and transform your vertex. There are two ways of achieving this:

* Convert straight to matrix and use the matrix to transform the vertex.
* Convert to quaternion/vector and transform the vertex using that.

The fastest and by far cleanest way is the second so I will concentrate on that. Given that you already have the rotation in the $Q_r$ component of the dual quaternion, extraction of the translation vector is achieved using the following:

$$V = 2Q_d*Q_r'$$

This can be implemented directly as:

~~~cpp
float4 Conjugate(float4 q)
{
	return float4(-q.x, -q.y, -q.z, q.w);
}

float4 Multiply(float4 a, float4 b)
{
	return float4(a.w * b.xyz + b.w * a.xyz + cross(b.xyz, a.xyz), a.w * b.w - dot(a.xyz, b.xyz));
}

float3 ReconstructTranslation(float4 Qr, float4 Qd)
{
	// The input is the dual quaternion, real part and dual part
	return Multiply(Qd, Conjugate(Qr)).xyz;
}
~~~

Of course, the complete calculation can be collapsed by directly applying the conjugate sign and discarding w:

~~~cpp
float3 ReconstructTranslation(float4 Qr, float4 Qd)
{
	return 2 * (Qr.w * Qd.xyz - Qd.w * Qr.xyz + cross(Qd.xyz, Qr.xyz));
}
~~~

Using the Conjugate and Multiply functions, it's then easy to transform a position and vector by the quaternion rotation and reconstructed position:

~~~cpp
float3 QuatRotateVector(float4 Qr, float3 v)
{
	// Straight-forward application of Q.v.Q', discarding w
	return Multiply(Multiply(Qr, float4(v, 0)), Conjugate(Qr)).xyz;
}

float3 DualQuatTransformPoint(float4 Qr, float4 Qd, float3 p)
{
	// Reconstruct translation from the dual quaternion
	float3 t = 2 * (Qr.w * Qd.xyz - Qd.w * Qr.xyz + cross(Qd.xyz, Qr.xyz));

	// Combine with rotation of the input point
	return QuatRotateVector(Qr, p) + t;
}
~~~

This leaves you with the final code:

~~~cpp
float2x4 skin_transform = BlendBoneTransforms(input.bone_indices, input.bone_weights);
float3 pos = DualQuatTransformPoint(skin_transform[0], skin_transform[1], input.pos);
float3 normal = QuatRotateVector(skin_transform[0], input.normal);
~~~


##### Optimising the Vertex Transformation


There's a bit of redundancy in the transformation code above; results being thrown away and inputs being used when they could be discarded. There are also some identities we can apply to the rotation equation that can simplify it. As it stands, reconstruction of the translation is good enough.

Starting with `QuatRotateVector`, we can already see that the first multiplication uses $w=0$, allowing us to construct a function which removes the necessary terms in its calculation:

~~~cpp
float4 MultiplyPure(float4 a, float3 b)
{
	return float4(a.w * b + cross(b, a.xyz), -dot(a.xyz, b));
}

float3 QuatRotateVector(float4 Qr, float3 v)
{
	return Multiply(MultiplyPure(Qr, v), Conjugate(Qr)).xyz;
}
~~~

The final redundancy is that we're calculating w and throwing it away, leading to:

~~~cpp
float3 MultiplyConjugate3(float4 a, float4 b)
{
	return b.w * a.xyz - a.w * b.xyz - cross(b.xyz, a.xyz);
}
float3 QuatRotateVector(float4 Qr, float3 v)
{
	return MultiplyConjugate3(MultiplyPure(Qr, v), Qr);
}
~~~

Realistically, the shader compiler should be able to handle all that for you. However, it gives us a good starting point to take this further.


##### We can do better than that


Let's try to explode the transformation and bring it back to something far simpler. I'll work through the steps I took in simplifying this explicitly - it serves as a nice record for me and will hopefully help if you're trying to understand where the final result came from (I was always losing signs during my school days - I'm no better 15 years on!)

We're trying to simplify:

$$P_r = Q*(0,V)*Q'$$

This is a sequence of two quaternion multiplies. Again, quaternion multiplication is defined as:

$$Q_0 Q_1 = (w_0 w_1 - V_0 \cdot V_1, w_0 V_1 + w_1 V_0 + V_0 \times V_1)$$

Let's make a few quick substitutions:

$$R = Q.xyz \\\ w = Q.w$$

Expand $Q*(0,V)$ first:

$$P_r = (-R \cdot V + wV + R \times V)(w - R)$$

Expand the second multiplication:

$$P_r = -R \cdot Vw + (wV + R \times V) \cdot R + (R \cdot V)R + w(wV + R \times V) - (wV + R \times V) \times R$$

The dot product distributes over addition so distribute them all:

$$P_r = -R \cdot Vw + wV \cdot R + R \times V \cdot R + (R \cdot V)R + + w^2V + wR \times V - (wV + R \times V) \times R$$

The first two terms cancel as the dot product is commutative:

$$P_r = R \times V \cdot R + (R \cdot V)R + w^2V + wR \times V - (wV + R \times V) \times R$$

Using the identity $A \times B=-B \times A$ swap the last cross product around:

$$P_r = R \times V \cdot R + (R \cdot V)R + w^2V + wR \times V + R \times (wV + R \times V)$$

The cross product distributes over addition so distribute the last cross product:

$$P_r = R \times V \cdot R + (R \cdot V)R + w^2V + wR \times V + R \times wV + R \times (R \times V)$$

As we're only interested in the `xyz` components of the result, discard all scalar terms:

$$P_r = (R \cdot V)R + w^2V + wR \times V + R \times wV + R \times (R \times V)$$

Pull the scalar out of $R \times wV$ and sum with its neighbour:

$$P_r = (R \cdot V)R + w^2V + wR \times V + wR \times V + R \times (R \times V)$$
$$P_r = (R \cdot V)R + w^2V + 2wR \times V + R \times (R \times V)$$

The next bit requires knowledge of the [vector triple product](http://en.wikipedia.org/wiki/Triple_product#Vector_triple_product) (or Lagrange's formula - of many). This takes the form:

$$R \times (R \times V) = (R \cdot V)R - (R \cdot R)V$$

If we rearrange that to equal zero then we can add that to the end of our existing equation and play around with it a little:

$$R \times (R \times V) - (R \cdot V)R + (R \cdot R)V = 0$$
$$P_r = (R \cdot V)R + w^2V + 2wR \times V + R \times (R \times V) + R \times (R \times V) - (R \cdot V)R + (R \cdot R)V$$
$$P_r = (R \cdot V)R + w^2V + 2wR \times V + 2R \times (R \times V) - (R \cdot V)R + (R \cdot R)V$$

The $(R \cdot V)R$ terms cancel:

$$P_r = w^2V + 2wR \times V + 2R \times (R \times V) + (R \cdot R)V$$

We can now factor the scale of $V$:

$$P_r = w^2V  + (R \cdot R)V+ 2wR \times V + 2R \times (R \times V)$$
$$P_r = (w^2  + R \cdot R)V+ 2wR \times V + 2R \times (R \times V)$$

The quaternion norm operation is given by:

$$norm(q) = q_w q_w + q_x q_x + q_y q_y + q_z q_z$$

Assuming we're dealing with unit quaternions, the norm will always be 1. Looking above, we can see the norm right at the beginning and can get rid of it:

$$P_r = V+ 2wR \times V + 2R \times (R \times V)$$

Finally, factor the 2:

$$P_r = V+ 2(wR \times V + R \times (R \times V))$$

And factor the cross product:

$$P_r = V+ 2(R \times (wV + R \times V))$$

This is a delightfully simple result! The HLSL code is:

~~~cpp
float3 QuatRotateVector(float4 Qr, float3 v)
{
	return v + 2 * cross(Qr.w * v + cross(v, Qr.xyz), Qr.xyz);
}
~~~
