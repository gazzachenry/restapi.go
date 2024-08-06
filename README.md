# restapi.go

package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/go-redis/redis/v8"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

// album represents data about a record album.
type album struct {
	ID     int64   `json:"id,omitempty" bson:"id,omitempty"`
	Title  string  `json:"title" bson:"title"`
	Artist string  `json:"artist" bson:"artist"`
	Price  float64 `json:"price" bson:"price"`
}

var (
	collection  *mongo.Collection
	redisClient *redis.Client
	mongoClient *mongo.Client
)

func main() {
	// Set MongoDB client options
	mongoClientOptions := options.Client().ApplyURI("mongodb://root:example@localhost:27017/")

	// Connect to MongoDB
	var err error
	mongoClient, err = mongo.Connect(context.TODO(), mongoClientOptions)
	if err != nil {
		log.Fatal(err)
	}

	// Check the connection
	err = mongoClient.Ping(context.TODO(), nil)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("Connected to MongoDB!")

	// Get the MongoDB collection
	collection = mongoClient.Database("hen").Collection("albums")
	if collection == nil {
		log.Fatal("Error getting collection")
	}

	// Set up Redis client
	redisClient = redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	defer redisClient.Close()

	// Check Redis connection
	_, err = redisClient.Ping(context.TODO()).Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Connected to Redis!")

	// Transfer data from MongoDB to Redis
	err = transferData()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Data transferred from MongoDB to Redis!")

	router := gin.Default()

	// Define the routes
	router.POST("/albums", postAlbums)
	router.GET("/albums", getAlbums)        // Add the GET method
	router.GET("/albums/:id", getAlbumByID) // New route for GET by ID
	router.PUT("/albums/:id", updateAlbum)  // New route for PUT
	router.DELETE("/albums/:id", deleteAlbum)

	// Run the Gin server
	router.Run("localhost:8080")
}

// transferData retrieves albums from MongoDB and stores them in Redis
func transferData() error {
	// Find all albums in MongoDB
	cursor, err := collection.Find(context.TODO(), bson.D{})
	if err != nil {
		return err
	}
	defer cursor.Close(context.TODO())

	for cursor.Next(context.TODO()) {
		var alb album
		if err := cursor.Decode(&alb); err != nil {
			return err
		}

		// Store album in Redis
		err := redisClient.Set(context.TODO(), fmt.Sprintf("album:%d", alb.ID), alb.Title, 0).Err()
		if err != nil {
			return err
		}
	}

	if err := cursor.Err(); err != nil {
		return err
	}

	return nil
}

// postAlbums adds an album from JSON received in the request body to MongoDB.
func postAlbums(c *gin.Context) {
	var newAlbum album

	// Bind the received JSON to newAlbum.
	if err := c.BindJSON(&newAlbum); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// Insert the new album into the MongoDB collection.
	_, err := collection.InsertOne(context.TODO(), newAlbum)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// Return the inserted album with the generated ID.
	// newAlbum.ID = result.InsertedID.(primitive.ObjectID)
	c.JSON(http.StatusCreated, newAlbum)
}

// getAlbums retrieves all albums from MongoDB and returns them as JSON.
func getAlbums(c *gin.Context) {
	var albums []album

	// Find all documents in the collection
	cursor, err := collection.Find(context.TODO(), bson.M{})
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	defer cursor.Close(context.TODO())

	// Iterate through the cursor and decode each document into an album
	for cursor.Next(context.TODO()) {
		var album album
		if err := cursor.Decode(&album); err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		albums = append(albums, album)
	}

	// Check for cursor errors
	if err := cursor.Err(); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// Return the albums as JSON
	c.JSON(http.StatusOK, albums)
}

// getAlbumByID responds with the album with the given ID as JSON.
func getAlbumByID(c *gin.Context) {
	// Get the album ID from the URL
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid album ID"})
		return
	}

	// Create a filter to find the album by ID
	filter := bson.M{"id": id}

	// Find the album from the MongoDB collection
	var album album
	err = collection.FindOne(context.TODO(), filter).Decode(&album)
	if err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "Album not found"})
		return
	}

	// Return the album as JSON
	c.JSON(http.StatusOK, album)
}

// deleteAlbum deletes an album by ID.
func deleteAlbum(c *gin.Context) {
	// Get the album ID from the URL
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid album ID"})
		return
	}

	// Create a filter to find the album by ID
	filter := bson.M{"id": id}

	// Delete the album from the MongoDB collection
	result, err := collection.DeleteOne(context.TODO(), filter)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// Check if the album was found and deleted
	if result.DeletedCount == 0 {
		c.JSON(http.StatusNotFound, gin.H{"error": "Album not found"})
		return
	}

	// Return a success message
	c.JSON(http.StatusOK, gin.H{"message": "Album deleted"})
}

// updateAlbum updates an album by ID.
func updateAlbum(c *gin.Context) {
	// Get the album ID from the URL
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid album ID"})
		return
	}

	// Bind the received JSON to newAlbum
	var updatedAlbum album
	if err := c.BindJSON(&updatedAlbum); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// Create a filter to find the album by ID
	filter := bson.M{"id": id}

	// Create an update document with the updated album details
	update := bson.M{
		"$set": updatedAlbum,
	}

	// Update the album in the MongoDB collection
	result, err := collection.UpdateOne(context.TODO(), filter, update)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// Check if the album was found and updated
	if result.MatchedCount == 0 {
		c.JSON(http.StatusNotFound, gin.H{"error": "Album not found"})
		return
	}

	// Return the updated album
	c.JSON(http.StatusOK, updatedAlbum)
}
