"""
Name: movie_recommendations.py
Date: 10/11/2020
Author: Tia Merheb and Julia Chambers
Description: 
Create a movie recommendation system which calculates the 
similarities of movies from the ratings given by users to 
the movies they have watched.
"""



import math
import csv
from scipy.stats import pearsonr

class Movie_Recommendations:
    def __init__(self, movie_filename, training_ratings_filename): #done
        """
        Initializes the Movie_Recommendations object from 
        the files containing movie names and training ratings.  
        The following instance variables should be initialized:
        self.movie_dict - A dictionary that maps a movie id to
               a movie objects (objects the class Movie)
        self.user_dict - A dictionary that maps user id's to a 
               a dictionary that maps a movie id to the rating
               that the user gave to the movie.    
        """
        #Initialize movie dictionary
        self.movie_dict = {}

        #open the file
        f = open(movie_filename)
        """"f.readline()"""
        csv_reader = csv.reader(f, delimiter = ',', quotechar = '"')
        next(csv_reader, None)
        for line in csv_reader:
            #construct movie object
            self.movie_dict[int(line[0])] = Movie(int(line[0]),line[1]) 
            #print(self.movie_dict[int(line[0])].title)

            
        #Initialize user dictionary
        self.user_dict = {}
        #Open file
        file = open(training_ratings_filename, "r")
        file.readline()
        ratings = []
        for line in file:
            ratings = line.split(",")
            if int(ratings[0]) not in self.user_dict.keys():
                self.user_dict[int(ratings[0])] = {int(ratings[1]):float(ratings[2])}
            else: 
                self.user_dict[int(ratings[0])].update({int(ratings[1]):float(ratings[2])})
            
            self.movie_dict[int(ratings[1])].users.append(int(ratings[0]))
        file.close()
        

    def predict_rating(self, user_id, movie_id): #need to do predicted rating and initialize ids?
        """
        Returns the predicted rating that user_id will give to the
        movie whose id is movie_id. 
        If user_id has already rated movie_id, return
        that rating.
        If either user_id or movie_id is not in the database,
        then BadInputError is raised."""
        

         #Initialize variables
        rating_dict = {}
        predicted_rating = 0
        similarity_calc = 0
        similarities = []
        similarity_rating_product = []

        #raise BadInputError if movie_id or user_id is not in the database
        if movie_id not in self.movie_dict or user_id not in self.user_dict:
            raise BadInputError
        else:
            if movie_id in self.user_dict[user_id]:
                rating_dict = self.user_dict[user_id]
                #if rating is already made by user, return
                predicted_rating = rating_dict[movie_id]
                return predicted_rating
            else:
                #define variables
                similarity_calc = self.movie_dict[movie_id]
                rating_dict = self.user_dict[user_id]

                for movie in rating_dict:
                    #call get_similarity function
                    similarity = similarity_calc.get_similarity(movie, self.movie_dict, self.user_dict)
                    #multiply the similarity by the rating
                    similarity_rating_product.append(similarity*rating_dict[movie])
                    similarities.append(similarity)

                if sum(similarities) == 0:
                    return 2.5 

                return sum(similarity_rating_product) / sum(similarities)

    def predict_ratings(self, test_ratings_filename):
        """
        Returns a list of tuples, one tuple for each rating in the
        test ratings file.
        The tuple should contain
        (user id, movie title, predicted rating, actual rating)
        """
        #open file
        f = open(test_ratings_filename)
        lines = f.readlines()
        prediction = 0
        title = 0
        list_of_ratings = []
        
        for line in lines[1:]:
            
            split = line.split(",")		
            prediction = self.predict_rating(int(split[0]), int(split[1]))
            #print(int(split[1]))
            #get movie title
            
            title = self.movie_dict[int(split[1])].title
            #create list of tuples
            list_of_ratings.append((int(split[0]), title, prediction, float(split[2])))
                            
        f.close()
        return list_of_ratings

    def correlation(self, predicted_ratings, actual_ratings): #done
        """
        Returns the correlation between the values in the list predicted_ratings
        and the list actual_ratings.  The lengths of predicted_ratings and
        actual_ratings must be the same.
        """
        return pearsonr(predicted_ratings, actual_ratings)[0]
        
class Movie: 
    """
    Represents a movie from the movie database.
    """
    def __init__(self, id, title): #done
        """ 
        Constructor.
        Initializes the following instances variables.  You
        must use exactly the same names for your instance 
        variables.  (For testing purposes.)
        id: the id of the movie
        title: the title of the movie
        users: list of the id's of the users who have
            rated this movie.  Initially, this is
            an empty list, but will be filled in
            as the training ratings file is read.
        similarities: a dictionary where the key is the
            id of another movie, and the value is the similarity
            between the "self" movie and the movie with that id.
            This dictionary is initially empty.  It is filled
            in "on demand", as the file containing test ratings
            is read, and ratings predictions are made.
        """
        #Initialize variables
        self.id = id
        self.title = title
        self.users = []
        self.similarities = {}


    def __str__(self): #done 
        """
        Returns string representation of the movie object.
        Handy for debugging.
        """
        #return title and id
        return self.title, self.id 

    def __repr__(self): #done 
        """
        Returns string representation of the movie object.
        """
        #return movie objects instance variables
        return self.id, self.title, self.users, self.similarities
    

    def get_similarity(self, other_movie_id, movie_dict, user_dict): #done?
        """ 
        Returns the similarity between the movie that 
        called the method (self), and another movie whose
        id is other_movie_id.  (Uses movie_dict and user_dict)
        If the similarity has already been computed, return it.
        If not, compute the similarity (using the compute_similarity
        method), and store it in both
        the "self" movie object, and the other_movie_id movie object.
        Then return that computed similarity.
        If other_movie_id is not valid, raise BadInputError exception.
        """
        #raise BadInputError if movie id is not found
        if other_movie_id not in movie_dict:
            raise BadInputError

        else: 
            #check if similarity has been computed and return 
            if other_movie_id in self.similarities:
                return self.similarities[other_movie_id]

            else:
                #if not, compute similarity and return
                return self.compute_similarity(other_movie_id, movie_dict, user_dict) 


    def compute_similarity(self, other_movie_id, movie_dict, user_dict): #compute similarity
        """ 
        Computes and returns the similarity between the movie that 
        called the method (self), and another movie whose
        id is other_movie_id.  (Uses movie_dict and user_dict)
        """
        #initialize variables
        other_movie = movie_dict[other_movie_id]
        users_who_watched_both = []
        rating_dict = {}
        movie1_rating = 0
        movie2_rating = 0
        difference_of_ratings = []
        average_diff = 0
        similarity = 0

        for user in self.users: 
            if user in other_movie.users:
                users_who_watched_both.append(user)
                rating_dict = user_dict[user]
                #assign the first and second ratings
                movie1_rating = rating_dict[self.id]
                movie2_rating = rating_dict[other_movie_id]
                #compute difference, add to list
                difference_of_ratings.append(abs(movie1_rating - movie2_rating))

        if len(users_who_watched_both) != 0:
            #calculate the average difference
            average_diff = sum(difference_of_ratings)/len(difference_of_ratings)
            #calculate the similarity
            similarity = - (average_diff/4.5) + 1
        else: 
            similarity = 0


        self.similarities[other_movie_id] = similarity
        
        # error here
        movie_dict[other_movie_id].similarities[self.id] = similarity

        return similarity

class BadInputError(Exception):
    pass

if __name__ == "__main__":
    # Create movie recommendations object.
    movie_recs = Movie_Recommendations("movies.csv", "training_ratings.csv")

    # Predict ratings for user/movie combinations
    rating_predictions = movie_recs.predict_ratings("test_ratings.csv")
    print("Rating predictions: ")
    for prediction in rating_predictions:
        print(prediction)
    predicted = [rating[2] for rating in rating_predictions]
    actual = [rating[3] for rating in rating_predictions]
    correlation = movie_recs.correlation(predicted, actual)
    print(f"Correlation: {correlation}") 
