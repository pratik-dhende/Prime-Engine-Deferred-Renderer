# Prime Engine Deferred Rendering Pipeline
- Implemented support for Deferred Rendering pipeline in the proprietary Prime Engine as a part of the CSCI 522 course, Milestone 1.
- Utilized RenderDoc for graphics debugging and profiling.

## Demo (Click on the image)
[![Deferred Rendering Demo Video](https://github.com/pratik-dhende/Prime-Engine-Deferred-Renderer/assets/55596801/7dace475-e97f-4c33-8203-5f35e91bc7ed)](https://drive.google.com/file/d/1lVPXfo9_IuJE9VypTJdOhNSaJ9uwIYwj/view?usp=sharing)

## Design diagram
![DeferredRenderingPipeline drawio (5)](https://github.com/pratik-dhende/Prime-Engine-Deferred-Renderer/assets/55596801/e13134de-8c7e-4b8e-8ce1-adc3d2dfc9ba)


## Implementation Approach
1.  Created a `DeferredRendering.fx` technique in `EffectManager::loadDefaultEffects` which uses `ColoredMinimalMesh_VS.cgvs` as it’s vertex shader and the newly created `ColoredMinimalMesh_DfrLightingPass_PS.cgps` as it’s pixel shader.
 
2.  Created a struct `GBuffer` which contains buffers for position, diffuse color, normal and shadow factor as `TextureGPU`. All of these are used for lighting calculation in the `ColoredMinimalMesh_DfrLightingPass_PS.cgps` for the deferred rendering lighting pass.
 
3. After the shadow map pass, the deferred rendering pass occurs.
	1. The G-Buffer render targets are set by:
		1. Creating screen size textures.
		2. Generating render target views from them.
		3. Getting a depth stencil view from one of the G-Buffers texture.
		4. Setting the render target views and depth stencil view.
		5. Creating a screen size viewport.
		6. Clearing the render target views and the depth stencil view.
	 
	2. Then Deferred Rendering Geometry pass happens:
		1. Effects which don’t require lighting are filtered out in the `DrawList::do_RENDER` the effect technique used.
		2. The shaders who require lighting like the `StdMesh_Shadowed_A_0_PS.cgps` and `DetailedMesh_Shadowed_A_Glow_PS.cgps` have been updated to output to the render tar gets by using the following struct in pixel shader. 
		```
		 struct DeferredRendering_PS_OUT { 
			 float4 positionW : SV_TARGET0; 
			 float4 diffusedColor : SV_TARGET1; 
			 float4 normalW : SV_TARGET2; 
			 float4 shadowFactor : SV_TARGET3; 
		 }; 
		```
			
	3. Then the Deferred Rendering Lighting pass happens:
		1.  `setTextureAndDepthTextureRenderTargetForGlow` is used to set the render target as this render target is then later used in the post processing pass.
		2. Then the full screen quad is created using `EffectManager::buildFullScreenBoard` is passed on to the GPU.
		3. Our G-Buffer render targets are bind to the pipeline as shader resource views using `SA_Bind_Resource::bindToPipeline`.
		4. Global shader constants for light calculations are bind to pipeline.
		5. All of the above steps are bind to the DeferredLighting.fx effect pipeline.
		6. The `ColoredMinimalMesh_DfrLightingPass_PS.cgps` samples the pixels from the G-Buffers, performs the lighting calculations, and gives the final color of the pixel.
		
	4. Then Forward rendering pass occurs:

		1. The final render target of the Deferred Rendering Lighting pass is again used as the rendering target for the forward rendering pass.
		2. The depth stencil view of the Deferred Rendering Lighting pass is again used as the depth stencil view for the forward rendering pass.
		3. Both the render target and depth stencil view is not cleared as the forward rendering pass need to render over the Deferred rendering lighting pass output.
		4. The `DrawList::do_RENDER` filters out the geometry which require lighting and thus only allows geometries like text and debug meshes to proceed with the usual forward rendering pass.
	
6. The final output of the Forward rendering pass contains all the geometries rendered with proper lighting.
7. This output is then further passed on for post processing after which the scene is finally shown.

## TODO
1. Optimize the memory usage by generating world space position through depth buffer.
2. Use tiling to improve performance.
