---
title: Structures generation
description: structures generation
---

```cs
using Godot;
[Tool, GlobalClass]
public partial class StructureType : Resource
{

        /// all should implement: IStructureShape
        [Export] public Resource[] shapes;
        [Export] public float maximal_height_delta_inside_the_shapes;
        [Export(PropertyHint.Range, "0,1,0.001")] public float spawn_chance;
        [Export] public PackedScene model;
        [Export] public float base_sale;
        [Export] public float scale_change_amplitude;
        [Export] public int min_distance_from_grid_border_in_mesh_chunks;
        [Export] public int generation_attempts_per_structure_chunk;

}
```

```cs
using System.Collections.Generic;
using Godot;

public interface IStructureShape
{
        public List<Vector2> GetSampleWorldPosPointsInsideTheShape(StructureInstanceData instance_data);
        public bool IsPointWithinTheShape(Vector2 point, StructureInstanceData instance_data);
}
```

```cs
using System.Collections.Generic;
using Godot;
[GlobalClass, Tool]
public partial class RectangleStructureShape : Resource, IStructureShape
{
        [Export] public Vector2 base_size;
        [Export] public float base_rotation_y;
        [Export] public Vector2 base_offset;
        [Export] public float sample_points_spacing;

        public List<Vector2> GetSampleWorldPosPointsInsideTheShape(StructureInstanceData instance_data)
        {
                var points = new List<Vector2>();

                // Get actual size and rotation from instance data
                Vector2 size = base_size * instance_data.scale;
                float rotation_rad = Mathf.DegToRad(base_rotation_y + instance_data.rotation_y);
                Vector2 offset = base_offset + instance_data.base_world_pos;

                // Calculate how many samples we need in each dimension
                int countX = Mathf.Max(1, Mathf.CeilToInt(size.X / sample_points_spacing));
                int countY = Mathf.Max(1, Mathf.CeilToInt(size.Y / sample_points_spacing));

                // Adjust spacing to fit evenly
                float actualSpacingX = size.X / countX;
                float actualSpacingY = size.Y / countY;

                // Generate points in local space
                for (int x = 0; x <= countX; x++)
                {
                        for (int y = 0; y <= countY; y++)
                        {
                                // Local point centered at origin
                                Vector2 localPoint = new(
                                    x * actualSpacingX - size.X / 2f,
                                    y * actualSpacingY - size.Y / 2f
                                );

                                // Apply rotation
                                Vector2 rotatedPoint = localPoint.Rotated(rotation_rad);

                                // Apply offset
                                points.Add(rotatedPoint + offset);
                        }
                }

                return points;
        }

        public bool IsPointWithinTheShape(Vector2 world_pos, StructureInstanceData instance_data)
        {
                // Get actual size and rotation from instance data
                Vector2 size = base_size * instance_data.scale;
                float rotation_rad = Mathf.DegToRad(base_rotation_y + instance_data.rotation_y);
                Vector2 offset = base_offset + instance_data.base_world_pos;

                // Transform point to local space (inverse of the rectangle's transform)
                Vector2 localPoint = (world_pos - offset).Rotated(-rotation_rad);

                // Check if point is within the axis-aligned bounds
                return Mathf.Abs(localPoint.X) <= size.X / 2f &&
                       Mathf.Abs(localPoint.Y) <= size.Y / 2f;
        }
}
```

```cs
using System.Collections.Generic;
using Godot;
public class StructureInstanceData(Vector2 base_world_pos, float scale, float rotation_y, StructureType structure_type)
{
        public Vector2 base_world_pos = base_world_pos;
        public float base_height;
        public float scale = scale;
        public float rotation_y = rotation_y;
        public StructureType structure_type = structure_type;

        public void Instantiate(Node3D parent)
        {
                var node = (Node3D)structure_type.model.Instantiate();
                parent.AddChild(node);
                node.Scale = Vector3.One * scale;
                node.RotationDegrees = new Vector3(0, rotation_y, 0);
                node.Position = new(base_world_pos.X, base_height, base_world_pos.Y);
        }

        public HashSet<Vector2I> MeshChunksThisStructureSitsOnWorldPos(int mesh_chunk_size)
        {
                HashSet<Vector2I> chunks = [];
                foreach (var shape in structure_type.shapes)
                {
                        foreach (var point in ((IStructureShape)shape).GetSampleWorldPosPointsInsideTheShape(this))
                        {

                                Vector2I chunk = new(Mathf.FloorToInt(point.X / mesh_chunk_size), Mathf.FloorToInt(point.Y / mesh_chunk_size));
                                Vector2I chunk_world_pos = chunk * mesh_chunk_size;
                                chunks.Add(chunk_world_pos);
                        }
                }
                return chunks;
        }
        public bool IsObjectColliding(Vector2I world_pos)
        {
                foreach (var shape in structure_type.shapes)
                {
                        if (((IStructureShape)shape).IsPointWithinTheShape(world_pos, this))
                                return true;
                }
                return false;
        }
        public List<Vector2> GetSampleWorldPosPointsInsideThisStructure()
        {
                List<Vector2> output = [];
                foreach (var shape in structure_type.shapes)
                {
                        output.AddRange(((IStructureShape)shape).GetSampleWorldPosPointsInsideTheShape(this));
                }

                return output;
        }

        public bool IsValid(GroundMeshGen mesh_gen)
        {
                var min_height = float.MaxValue;
                var max_height = float.MinValue;
                foreach (var shape in structure_type.shapes)
                {
                        var i_structure_shape = (IStructureShape)shape;
                        var test_points = i_structure_shape.GetSampleWorldPosPointsInsideTheShape(this);
                        foreach (var point in test_points)
                        {
                                var height = mesh_gen.CalculateHeight(point, out _);
                                min_height = Mathf.Min(height, min_height);
                                max_height = Mathf.Max(height, max_height);
                        }
                        if (max_height - min_height > structure_type.maximal_height_delta_inside_the_shapes)
                                return false;
                }

                return true;
        }

}
```

```cs
using System.Collections.Generic;
using Godot;
[Tool]
public partial class StructureGen : Node
{
        [Export] StructureType[] structure_pool;
        [Export] public int mesh_chunks_per_structure_grid_cell;

        public class StructureChunk(Dictionary<Vector2I, StructureInstanceData> structures_for_mesh_chunk_world_pos, Dictionary<Vector2I, StructureInstanceData> structure_gen_for_mesh_chunk_world_pos)
        {
                public Dictionary<Vector2I, StructureInstanceData> structures_for_mesh_chunk_world_pos = structures_for_mesh_chunk_world_pos;
                public Dictionary<Vector2I, StructureInstanceData> structure_gen_for_mesh_chunk_world_pos = structure_gen_for_mesh_chunk_world_pos;
        }
        public class StructureGrid
        {
                const int grid_width = 3;
                readonly StructureChunk[] grid;
                private Vector2I current_player_grid_pos;

                private readonly int grid_cell_size;
                private readonly StructureGen structure_gen;
                private readonly GroundMeshGen mesh_gen;
                private readonly int mesh_chunk_size;
                public bool IsObjectValid(Vector2 world_pos_f)
                {
                        Vector2I world_pos = (Vector2I)world_pos_f;
                        var chunk_pos = world_pos / mesh_chunk_size;
                        var chunk_world_pos = chunk_pos * mesh_chunk_size;



                        if (!this[world_pos].structures_for_mesh_chunk_world_pos.TryGetValue(chunk_world_pos, out var structure))
                                return true;

                        return !structure.IsObjectColliding(world_pos);
                }
                public StructureChunk this[Vector2I world_pos]
                {
                        get
                        {
                                var global_grid_pos = world_pos / grid_cell_size;
                                var relative_grid_pos = global_grid_pos - current_player_grid_pos;
                                return grid[relative_grid_pos.X + 1 + (relative_grid_pos.Y + 1) * grid_width];
                        }

                }

                public void UpdatePlayerPos(Vector2 player_world_pos)
                {
                        var new_player_grid_pos = new Vector2I((int)player_world_pos.X / grid_cell_size, (int)player_world_pos.Y / grid_cell_size);
                        var delta = current_player_grid_pos - new_player_grid_pos;
                        if (delta == Vector2I.Zero)
                        {
                                return;
                        }
                        current_player_grid_pos = new_player_grid_pos;

                        // this is expensive, but allows for really fast access of the data 
                        var grid_copy = (StructureChunk[])grid.Clone();
                        for (int x = 0; x < grid_width; x++)
                        {
                                for (int y = 0; y < grid_width; y++)
                                {
                                        var new_x = x - delta.X;
                                        var new_y = y - delta.Y;
                                        if (new_x < 0 || new_y < 0 || new_x >= grid_width || new_y >= grid_width)
                                        {
                                                return;
                                        }
                                        if (new_x == 1 || new_y == 1)
                                        {
                                                //Generate new cell data
                                                var world_x = (x - 1 + new_player_grid_pos.X) * grid_cell_size;
                                                var world_y = (y - 1 + new_player_grid_pos.Y) * grid_cell_size;
                                                grid[x + y * grid_width] = GenerateChunk(new(world_x, world_y));
                                        }

                                        grid[new_x + new_y * grid_width] = grid_copy[x + y * grid_width];
                                }
                        }

                }

                public StructureGrid(StructureGen structure_gen, GroundMeshGen mesh_gen, int mesh_chunk_size, Vector2 player_world_pos)
                {
                        grid_cell_size = structure_gen.mesh_chunks_per_structure_grid_cell * mesh_chunk_size;
                        this.structure_gen = structure_gen;
                        this.mesh_gen = mesh_gen;
                        this.mesh_chunk_size = mesh_chunk_size;

                        current_player_grid_pos = new Vector2I((int)player_world_pos.X / grid_cell_size, (int)player_world_pos.Y / grid_cell_size);
                        grid = new StructureChunk[grid_width * grid_width];
                        for (int x = 0; x < grid_width; x++)
                        {
                                for (int y = 0; y < grid_width; y++)
                                {
                                        //Generate new cell data
                                        var world_x = (x - 1 + current_player_grid_pos.X) * grid_cell_size;
                                        var world_y = (y - 1 + current_player_grid_pos.Y) * grid_cell_size;
                                        grid[x + y * grid_width] = GenerateChunk(new(world_x, world_y));
                                }
                        }
                }

                public StructureChunk GenerateChunk(Vector2I base_world_pos)
                {
                        Dictionary<Vector2I, StructureInstanceData> structure_gen_for_mesh_chunk_world_pos = [];
                        Dictionary<Vector2I, StructureInstanceData> structures_for_mesh_chunk_world_pos = [];
                        RNG.SetGDRandomSeed(base_world_pos);
                        foreach (var structure_type in structure_gen.structure_pool)
                        {

                                for (int i = 0; i < structure_type.generation_attempts_per_structure_chunk; i++)
                                {

                                        if (structure_type.spawn_chance < GD.Randf())
                                                continue;
                                        if (structure_gen.mesh_chunks_per_structure_grid_cell < 2 * structure_type.min_distance_from_grid_border_in_mesh_chunks)
                                                GD.PrintErr("structure_gen.mesh_chunks_per_structure_grid_cell has to ge at least 2x the structure_type.min_distance_from_grid_border_in_mesh_chunks.");


                                        var mesh_chunk_x = GD.RandRange(structure_type.min_distance_from_grid_border_in_mesh_chunks, structure_gen.mesh_chunks_per_structure_grid_cell - structure_type.min_distance_from_grid_border_in_mesh_chunks);
                                        var mesh_chunk_y = GD.RandRange(structure_type.min_distance_from_grid_border_in_mesh_chunks, structure_gen.mesh_chunks_per_structure_grid_cell - structure_type.min_distance_from_grid_border_in_mesh_chunks);
                                        var base_chunk_world_pos = new Vector2I(mesh_chunk_x, mesh_chunk_y) * mesh_chunk_size + base_world_pos;

                                        if (structures_for_mesh_chunk_world_pos.ContainsKey(base_chunk_world_pos))
                                                continue;

                                        var structure_world_pos = new Vector2(GD.Randf(), GD.Randf()) * mesh_chunk_size + base_chunk_world_pos;
                                        var structure_rotation = GD.Randf() * 360f;
                                        var structure_scale = structure_type.base_sale + (GD.Randf() * 2 - 1) * structure_type.scale_change_amplitude;
                                        var structure_instance = new StructureInstanceData(structure_world_pos, structure_scale, structure_rotation, structure_type);
                                        if (!structure_instance.IsValid(mesh_gen))
                                                continue;
                                        structure_instance.base_height = mesh_gen.CalculateHeight(structure_world_pos, out _);

                                        bool there_already_was_struct_on_one_of_the_chunks = false;

                                        var all_chunks = structure_instance.MeshChunksThisStructureSitsOnWorldPos(mesh_chunk_size);

                                        foreach (var chunk_world_pos in all_chunks)
                                        {
                                                if (structures_for_mesh_chunk_world_pos.ContainsKey(chunk_world_pos))
                                                {
                                                        there_already_was_struct_on_one_of_the_chunks = true;
                                                        break;
                                                }
                                        }
                                        if (there_already_was_struct_on_one_of_the_chunks)
                                                continue;


                                        foreach (var chunk_world_pos in all_chunks)
                                        {
                                                structures_for_mesh_chunk_world_pos.Add(chunk_world_pos, structure_instance);
                                        }

                                        structure_gen_for_mesh_chunk_world_pos.Add(base_chunk_world_pos, structure_instance);

                                }
                        }
                        return new(structures_for_mesh_chunk_world_pos, structure_gen_for_mesh_chunk_world_pos);

                }
        }
}
```

```diff lang="cs"
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Godot;
[Tool]
public partial class GenerationController : Node
{
        [ExportToolButton("Run")] private Callable RunButton => Callable.From(RunClean);
        [Export] int terrain_chunk_size;
        [Export] Biome[] biomes;
        [Export] int max_main_thread_chunk_instantiation_per_frame;

        [ExportGroup("player")]
        [Export] Vector2 player_pos_offset;
        [Export] Node3D player;
        [Export] int view_distance_chunks;

        [ExportCategory("references")]
        [Export] PackedScene chunk_prefab;
        [Export] GroundMeshGen ground_mesh_gen;
        [Export] BiomeGenerator biome_generator;
        [Export] GroundShaderController ground_shader_controller;
        [Export] ObjectsGenerator objects_generator;
+       [Export] StructureGen structure_gen;
+       StructureGen.StructureGrid structure_grid;

        public override void _Ready()
        {
                if (!Engine.IsEditorHint())
                        RunClean();
        }

        /// When you want to change you need to also change the value in the ground shader 
        const int max_chunk_data_textures_count = 517;
        public override void _Process(double delta)
        {
                ChunkDataGeneration();
                InstantiateChunksFromQue();
        }
        private void DestroyChunks(Vector2I[] chunks_to_destroy)
        {
                foreach (var chunk_relative_pos in chunks_to_destroy)
                {
                        Vector2I chunk_world_position = chunk_relative_pos + last_player_chunk_grid_pos * terrain_chunk_size;

                        if (!chunk_per_world_position.TryGetValue(chunk_world_position, out var chunk))
                        {
                                GD.PushWarning("There was already a chunk in the dictionary at this position. This either indicates a but in the logic of this program or the player did some crazy movements. Regenerating the whole terrain.");

                                ClearAll();
                                GenerateDataForAllChunks();
                                return;
                        }

                        free_biome_texture_slots.Enqueue(chunk.biome_map_index);
                        chunk.QueueFree();
                        chunk_per_world_position.Remove(chunk_world_position);
                }

        }

        private void RunClean()
        {
                ClearAll();

                ChunkChangeCalculator.Init(view_distance_chunks, terrain_chunk_size);

                if (max_chunk_data_textures_count != ChunkChangeCalculator.GetAllChunksInViewDistance().Count)
                {
                        GD.PushWarning("The max amount of chunk data textures is not equal to the chunk data textures that are generated.\n" +
                                "This is not optimal and could cause chunks biomes to stop working:\n" +
                                $"current:{max_chunk_data_textures_count} optimal:{ChunkChangeCalculator.GetAllChunksInViewDistance().Count}");
                }

                ground_mesh_gen.Initialize(terrain_chunk_size);
                ground_shader_controller.SetShaderConfiguration(biomes);
+                structure_grid = new StructureGen.StructureGrid(structure_gen, ground_mesh_gen, terrain_chunk_size, new(player.Position.X, player.Position.Z));

                GenerateDataForAllChunks();
        }

+-      public struct ChunkData(GroundMeshGen.MeshData mesh_data, BiomeGenerator.TextureData biome, Vector2I world_pos, ObjectsGenerator.ObjectTypeSpawnData[] objects_data)
+       public struct ChunkData(GroundMeshGen.MeshData mesh_data, BiomeGenerator.TextureData biome, Vector2I world_pos, ObjectsGenerator.ObjectTypeSpawnData[] objects_data, StructureInstanceData? structure)
        {
                public ObjectsGenerator.ObjectTypeSpawnData[] objects_data = objects_data;
                public GroundMeshGen.MeshData mesh_data = mesh_data;
                public BiomeGenerator.TextureData biome = biome;
                public Vector2I world_pos = world_pos;
+               public StructureInstanceData? structure = structure;
        }

        Vector2I WorldToTerrainChunkGridPos(Vector2 world_pos)
        {
                return new Vector2I(Mathf.RoundToInt(world_pos.X / terrain_chunk_size), Mathf.RoundToInt(world_pos.Y / terrain_chunk_size));
        }
        Dictionary<Vector2I, Chunk> chunk_per_world_position;

        Vector2I last_player_chunk_grid_pos;
        private void ChunkDataGeneration()
        {

                // This could happen after building the project in the godot editor while the generation  process is running
                if (chunk_data_generation_task == null)
                {
                        ClearAll();
                        GenerateDataForAllChunks();
                }

                if (!chunk_data_generation_task.IsCompleted || !chunk_instantiation_que.IsEmpty)
                {
                        return;
                }

                // Update only once all chunks from the previous batch have been generated / destroyed  
                ground_shader_controller.UpdateTheBiomeTextures(biome_textures_channel_1, biome_textures_channel_2);

                Vector2 player_pos = new(player.Position.X, player.Position.Z);

                var current_player_chunk_grid_pos = WorldToTerrainChunkGridPos(player_pos);

                if (last_player_chunk_grid_pos == current_player_chunk_grid_pos)
                {
                        return;
                }
                var grid_pos_delta = current_player_chunk_grid_pos - last_player_chunk_grid_pos;

                if (!ChunkChangeCalculator.chunk_change_for_position_delta.TryGetValue(grid_pos_delta, out var chunk_change))
                {
                        GD.PushWarning("The position of player changed by more than a 1 chunk size which is not supported. Regenerating the whole terrain.");

                        last_player_chunk_grid_pos = current_player_chunk_grid_pos;
                        ClearAll();
                        GenerateDataForAllChunks();
                        return;
                }


+               structure_grid.UpdatePlayerPos(player_pos);
                DestroyChunks(chunk_change.to_destroy_relative_pos);
                GenerateDataForChunks(chunk_change.to_generate_relative_pos, current_player_chunk_grid_pos * terrain_chunk_size);
                last_player_chunk_grid_pos = current_player_chunk_grid_pos;
        }
        private void GenerateDataForAllChunks()
        {
                var chunks_to_generate = ChunkChangeCalculator.GetAllChunksInViewDistance();
                GenerateDataForChunks([.. chunks_to_generate], last_player_chunk_grid_pos * terrain_chunk_size);
        }

        Task chunk_data_generation_task;
        ConcurrentQueue<ChunkData> chunk_instantiation_que = new();
        private void GenerateDataForChunks(Vector2I[] chunks_to_generate, Vector2I player_pos_snapped_to_chunk)
        {
                chunk_data_generation_task = Parallel.ForEachAsync(Enumerable.Range(0, chunks_to_generate.Length), async (i, _) =>
                     {
                             try
                             {
                                     var chunk = chunks_to_generate[i];
                                     Vector2I chunk_world_position = chunk + player_pos_snapped_to_chunk;

                                     var biome_data = biome_generator.GenerateTextureData(chunk_world_position, terrain_chunk_size + 1, biomes);
                                     var objects_data = objects_generator.GenerateObjectsData(terrain_chunk_size, biome_data, chunk_world_position);
                                     var mesh_data = ground_mesh_gen.GenerateChunkData(chunk_world_position);
+                                    structure_grid[chunk_world_position].structure_gen_for_mesh_chunk_world_pos.TryGetValue(chunk_world_position, out var chunk_structure_data);



-                                    chunk_instantiation_que.Enqueue(new(mesh_data, biome_data, chunk_world_position, objects_data));
+                                    chunk_instantiation_que.Enqueue(new(mesh_data, biome_data, chunk_world_position, objects_data, chunk_structure_data));
                             }
                             catch (Exception e)
                             {
                                     GD.PrintErr($"GenerateDataForChunks failed: {e}");
                             }
                     });
        }
        private void InstantiateChunksFromQue()
        {
                int processed = 0;
                while (processed < max_main_thread_chunk_instantiation_per_frame && chunk_instantiation_que.TryDequeue(out var chunk_data))
                {
                        InstantiateChunk(chunk_data);
                        processed++;
                }
        }

        Queue<int> free_biome_texture_slots;
        ImageTexture[] biome_textures_channel_1;
        ImageTexture[] biome_textures_channel_2;
        private void InstantiateChunk(ChunkData chunk_data)
        {

                var chunk = (Chunk)chunk_prefab.Instantiate();
                chunk_per_world_position.Add(chunk_data.world_pos, chunk);

                AddChild(chunk);
                ground_mesh_gen.ApplyData(chunk_data.mesh_data, chunk.mesh_instance, chunk.collider);
                ObjectsGenerator.SpawnObjects(chunk_data.objects_data, chunk);


                int map_index = free_biome_texture_slots.Dequeue();
                chunk.biome_map_index = map_index;
                biome_textures_channel_1[map_index] = chunk_data.biome.GetTexture(0);
                biome_textures_channel_2[map_index] = chunk_data.biome.GetTexture(1);
                chunk.mesh_instance.SetInstanceShaderParameter("biome_texture_index", map_index);
                chunk_data.structure?.Instantiate(chunk);
        }
        private void ClearAll()
        {
                free_biome_texture_slots = new(Enumerable.Range(0, max_chunk_data_textures_count));
                biome_textures_channel_1 = new ImageTexture[max_chunk_data_textures_count];

                biome_textures_channel_2 = new ImageTexture[max_chunk_data_textures_count];

                chunk_per_world_position = [];

                foreach (var child in GetChildren())
                {
                        child.QueueFree();
                }
        }
}
```

#### Bugs

If you find anything to improve in this project's code, please create an issue
describing it on the
[GitHub repository for this project](https://github.com/FilipRuman/procedural_terrain_generationV2/issues).
For website-related issues, create an issue
[here](https://github.com/FilipRuman/pages/issues).

#### Support

All pages on this site are written by a human, and you can access everything for
free without ads. If you find this work valuable, please give a star to the
[GitHub repository for this project](https://github.com/FilipRuman/procedural_terrain_generationV2).

<script src="https://giscus.app/client.js"
        data-repo="FilipRuman/procedural_terrain_generationV2"
        data-repo-id="R_kgDOQlnCIA"
        data-category="Announcements"
        data-category-id="DIC_kwDOQlnCIM4C4CHB"
        data-mapping="specific"
        data-term="structures generation"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="en"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>
