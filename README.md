**using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Tilemaps;

public class WorldBuilder2D : MonoBehaviour
{
    [Header("Terrain Generation")]
    public Tilemap terrainTilemap;
    public Tilemap grassTilemap;
    public List<TileBase> groundTiles;
    public List<TileBase> grassTiles;

    [Header("World Settings")]
    public int width = 100;
    public int height = 20;
    public float noiseScale = 0.1f;
    public float groundHeight = 12f;
    public int seed;

    [Header("Player")]
    public Transform playerTransform;
    public float generationDistance = 10f;

    public float[,] noiseMap { get; private set; }

    private float[] heightMap;
    private Vector2 lastPlayerPosition;
    private bool isHeightMapGenerated;

    private void Start()
    {
        lastPlayerPosition = GetPlayerPosition();
        GenerateHeightMap();
        GenerateTerrain();
    }

    private void Update()
    {
        Vector2 currentPlayerPosition = GetPlayerPosition();

        if (Mathf.Abs(currentPlayerPosition.y - lastPlayerPosition.y) >= generationDistance ||
            Mathf.Abs(currentPlayerPosition.x - lastPlayerPosition.x) >= generationDistance)
        {
            lastPlayerPosition = currentPlayerPosition;
            GenerateTerrain();
        }
    }

    private void GenerateTerrain()
    {
        if (!isHeightMapGenerated)
            return;

        int startX = Mathf.FloorToInt(playerTransform.position.x - generationDistance);
        int endX = Mathf.CeilToInt(playerTransform.position.x + generationDistance);

        terrainTilemap.ClearAllTiles();
        grassTilemap.ClearAllTiles();

        GenerateTerrainChunk(startX, endX);
    }

    private void GenerateTerrainChunk(int startX, int endX)
    {
        startX = Mathf.Clamp(startX, 0, width - 1);
        endX = Mathf.Clamp(endX, 0, width - 1);

        for (int x = startX; x <= endX; x++)
        {
            int currentHeight = Mathf.RoundToInt(heightMap[x]);
            int previousHeight = currentHeight;

            for (int y = 0; y <= currentHeight; y++)
            {
                SetTileAtPosition(x, y, groundTiles, terrainTilemap);

                if (y < currentHeight && y < height - 1 && !IsSolidTile(x, y + 1))
                {
                    SetTileAtPosition(x, y + 1, grassTiles, grassTilemap);
                }
            }
        }
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

    public bool IsCaveTile(int x, int y)
    {
        // Define your cave tile logic here
        // ...

        return false;
    }
}
**
