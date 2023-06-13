# Continuous-procedurall-2D-Terrain
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Tilemaps;

[System.Serializable]
public struct OreTileMapping
{
    public Tilemap oreTilemap;
    public List<TileBase> tiles;
    public float threshold;
    public Color color;
    public int seed;
}

public class WorldBuilder2D : MonoBehaviour
{
    [Header("Terrain Generation")]
    public Tilemap terrainTilemap;
    public Tilemap grassTilemap;
    public List<TileBase> groundTiles;
    public List<TileBase> waterTiles;
    public List<TileBase> grassTiles;

    [Header("Ore Generation")]
    public List<OreTileMapping> oreTiles;

    [Header("World Settings")]
    public int width = 100;
    public int height = 20;
    public float noiseScale = 0.1f;
    public float groundHeight = 12f;
    public float waterHeight = 6f;
    public int seed;

    [Header("Player")]
    public Transform playerTransform;
    public float generationDistance = 10f;

    [Header("Cave Generation")]
    public float caveThreshold = 0.5f;

    private float[,] noiseMap;
    private float[] heightMap;
    private bool[,] caveTiles;
    private Vector2 lastPlayerPosition;
    private bool isHeightMapGenerated;

    private void Start()
    {
        lastPlayerPosition = GetPlayerPosition();
        GenerateHeightMap();
        GenerateCaveTexture();
        GenerateTerrain();
        GenerateOre();
    }

    private void Update()
    {
        Vector2 currentPlayerPosition = GetPlayerPosition();

        if (Mathf.Abs(currentPlayerPosition.y - lastPlayerPosition.y) >= generationDistance ||
            Mathf.Abs(currentPlayerPosition.x - lastPlayerPosition.x) >= generationDistance)
        {
            lastPlayerPosition = currentPlayerPosition;
            GenerateTerrain();
            GenerateOre();
        }
    }

    private void GenerateTerrain()
    {
        if (!isHeightMapGenerated)
            return;

        float spawnY = Mathf.Clamp(playerTransform.position.y - generationDistance, 0f, height - 1);
        float spawnX = Mathf.Clamp(playerTransform.position.x - generationDistance, 0f, width - 1);
        float despawnY = Mathf.Clamp(playerTransform.position.y + generationDistance, 0f, height - 1);
        float despawnX = Mathf.Clamp(playerTransform.position.x + generationDistance, 0f, width - 1);

        terrainTilemap.ClearAllTiles();
        grassTilemap.ClearAllTiles();

        int startX = Mathf.FloorToInt(spawnX);
        int startY = Mathf.FloorToInt(spawnY);
        int endX = Mathf.CeilToInt(despawnX);
        int endY = Mathf.CeilToInt(despawnY);

        // Generate terrain on the X-axis from spawnX to despawnX
        for (int x = startX; x <= endX; x++)
        {
            for (int y = startY; y <= endY; y++)
            {
                if (y <= heightMap[x])
                {
                    SetTileAtPosition(x, y, groundTiles, terrainTilemap);

                    if (y < heightMap[x] && y < height - 1 && !IsSolidTile(x, y + 1))
                    {
                        SetTileAtPosition(x, y + 1, grassTiles, grassTilemap);
                    }
                }
                else if (y <= waterHeight)
                {
                    SetTileAtPosition(x, y, waterTiles, terrainTilemap);
                }
            }
        }

        // Generate terrain on the X-axis from despawnX back to spawnX
        for (int x = endX - 1; x >= startX; x--)
        {
            for (int y = startY; y <= endY; y++)
            {
                if (y <= heightMap[x])
                {
                    SetTileAtPosition(x, y, groundTiles, terrainTilemap);

                    if (y < heightMap[x] && y < height - 1 && !IsSolidTile(x, y + 1))
                    {
                        SetTileAtPosition(x, y + 1, grassTiles, grassTilemap);
                    }
                }
                else if (y <= waterHeight)
                {
                    SetTileAtPosition(x, y, waterTiles, terrainTilemap);
                }
            }
        }
    }

    private void GenerateOre()
    {
        foreach (var ore in oreTiles)
        {
            if (ore.oreTilemap == null)
                continue;

            ore.oreTilemap.ClearAllTiles();

            Random.InitState(ore.seed);

            for (int x = 0; x < width; x++)
            {
                for (int y = 0; y < height; y++)
                {
                    if (IsCaveTile(x, y))
                    {
                        float oreNoise = noiseMap[x, y];

                        if (oreNoise < ore.threshold)
                        {
                            SetTileAtPosition(x, y, ore.tiles, ore.oreTilemap);
                            ore.oreTilemap.SetColor(new Vector3Int(x, y, 0), ore.color);
                        }
                    }
                }
            }
        }
    }

    private void GenerateHeightMap()
    {
        heightMap = new float[width];
        noiseMap = PerlinNoiseGenerator.GenerateNoiseMap(width, 1, noiseScale, seed);

        for (int i = 0; i < width; i++)
        {
            float noiseValue = noiseMap[i, 0];
            heightMap[i] = Mathf.Lerp(0f, groundHeight, noiseValue);
        }

        isHeightMapGenerated = true;
    }

    private void GenerateCaveTexture()
    {
        caveTiles = new bool[width, height];

        noiseMap = PerlinNoiseGenerator.GenerateNoiseMap(width, height, noiseScale, seed);

        for (int x = 0; x < width; x++)
        {
            for (int y = 0; y < height; y++)
            {
                float noiseValue = noiseMap[x, y];

                if (noiseValue < caveThreshold)
                {
                    caveTiles[x, y] = true;
                }
                else
                {
                    caveTiles[x, y] = false;
                }
            }
        }
    }

    private bool IsCaveTile(int x, int y)
    {
        return caveTiles[x, y];
    }

    private bool IsSolidTile(int x, int y)
    {
        if (x < 0 || x >= width || y < 0 || y >= height)
            return false;

        return heightMap[x] >= y;
    }

    private void SetTileAtPosition(int x, int y, List<TileBase> tiles, Tilemap tilemap)
    {
        if (tiles == null || tiles.Count == 0)
            return;

        if (x < 0 || x >= width || y < 0 || y >= height)
            return;

        int randomIndex = Random.Range(0, tiles.Count);
        TileBase tile = tiles[randomIndex];
        tilemap.SetTile(new Vector3Int(x, y, 0), tile);
    }

    private Vector2 GetPlayerPosition()
    {
        return new Vector2(playerTransform.position.x, playerTransform.position.y);
    }
}
