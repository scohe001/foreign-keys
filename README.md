# Foreign Key Relationships

### Who will this be useful for?

This guide will be useful for you if you...

- Are still in the process of setting up your backend (database and models) for your Capstone
- You want the user to be able to enter items that will then later be used in a dropdown and stored as part of another object
- You want to declare ownership (ie: each user has a list of Meals they've created)
- You have a many-to-many relationship (ie: each Meal has many Ingredients from the same original list of ingredients)

For more on dotnet db relationships, you can also see the [the Microsoft docs](https://docs.microsoft.com/en-us/ef/core/modeling/relationships) (click the "Data Annotations" tab on examples where applicable to see how you should modify your Model code).

## Basic Foreign Key relationship

Foreign Keys help us link data from two separate tables. In my example capstone, I'm building meals. To ensure a meal is healthy, I want every meal to have exactly one source of protein. However, everyone uses different things as protein sources, so I want the user to be able to enter in their own protein sources and then when building a meal, choose from a dropdown of all of the proteins entered.

I could just have the user enter a string for the protein when creating a meal, but I like the idea of having a dropdown for them to pick from (and this also lets me store things on the protein like CaloriesPerOunce, which I can display alongside the proteins in the dropdown!).

So I'll need a protein table to store all the possible proteins:

| Id |   Title | CaloriesPerOunce |
|----|---------|------------------|
| 1  | Chicken | 35               |
| 2  | Beef    | 50               |
| 3  | Tofu    | 20               |

In Model code, this would look like:

    using System;
    using System.Collections.Generic;
    using System.ComponentModel.DataAnnotations;
    using System.ComponentModel.DataAnnotations.Schema;
    using System.Linq;
    using System.Threading.Tasks;
      
    namespace capstone.Models
    {
        public class Protein
        { 
            [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
            [Key]
            public int Id { get; set; }
            
            public string Title { get; set; }
            
            public int CaloriesPerOunce { get; set; }
        }
    }

Now for each meal, I want it to reference a protein that it'll use. This is where foreign keys come in!!

For example, if my Meal table looks like:

| Id |   Title | ProteinId | Total Calories |
|----|---------|--------|----------------|
| 1  | Tofu Stir Fry | 3 | 150 |
| 2  | Chicken Salad | 1 | 200 |

That `ProteinId` column is actually a foreign key or "reference" to the Protein table! So it's saying that Tofu Stir Fry uses `ProteinId` 3, which, looking at the Protein table, is tofu!

If we have a foreign key relationship here, we'll also have some safety in our database. If I try to insert a new meal into the Meal table with a bad `ProteinId` (one that doesn't exist in the Protein table), SQLite simply won't let me! So you'll have some built in safety if your users manage to bypass your frontend and send you bad data.

And the coolest part here is that in the C# code, you don't even have to play with Id's, all you see is that Meal has a Protein data member:

    using System;
    using System.Collections.Generic;
    using System.ComponentModel.DataAnnotations;
    using System.ComponentModel.DataAnnotations.Schema;
    using System.Linq;
    using System.Threading.Tasks;
      
    namespace capstone.Models
    {
        public class Meal
        {
            [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
            [Key]
            public int Id { get; set; }
            
            public string Title { get; set; }
            
            public int Calories { get; set; }
            
            // The actual Id the table is storing. This is the foregin key!
            public int ProteinId { get; set; } 

            [ForeignKey("ProteinId")] // <-- Needs to match the name of the ForeignKey property
            public Protein ProteinItem { get; set; }
        }
    }

If you get this far, I'd try for a migration right now. When you open the migration file to check it (which you should do before you update your database!), you should look for the `Up()` function. This is the changes that are about to be applied to your database (`Down()` would be the logic that gets run if you rollback this migration). Your `Up()` should look something like:

    migrationBuilder.CreateTable(
        name: "Proteins",
        columns: table => new
        {
            Id = table.Column<int>(nullable: false)
            .Annotation("Sqlite:Autoincrement", true),
            Title = table.Column<string>(nullable: true)
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Proteins", x => x.Id);
        });

    migrationBuilder.CreateTable(
        name: "Meals",
        columns: table => new
        {
            Id = table.Column<int>(nullable: false)
            .Annotation("Sqlite:Autoincrement", true),
            Title = table.Column<string>(nullable: true),
            ProteinId = table.Column<int>(nullable: true),
            Calories = table.Column<int>(nullable: false)
        
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Meals", x => x.Id);
            table.ForeignKey(
            name: "FK_Meals_Proteins_ProteinId",
            column: x => x.ProteinId,
            principalTable: "Proteins",
            principalColumn: "Id",
            onDelete: ReferentialAction.Restrict);
        });

Notice how `ProteinItem` isn't going to be part of the table at all. All the table holds is the foreign key `ProteinId`, but our ORM will automagically transform that into a Protein item for us to use in C#!

## One-to-Many relationship

Let's say I want each meal to **belong to** a user. So I want each user to have a list of meals.

To link a meal to a user, I'll add a foreign key to my meal:

    // The actual Id the table is storing. This is the foregin key!
    public int OwnerId { get; set; } 

    [ForeignKey("OwnerId")] // <-- Needs to match the name of the ForeignKey property
    public User Owner { get; set; }

You may be thinking "But this is the same thing we did with meals and proteins, isn't it?" And you're correct! We're linking the same way, but semantically we're trying to say something entirely different. Before we meant that every meal *had* a single protein from the list. But now we're trying to say that every meal *belongs* to a user. And because of this different relationship, we want to be able to access a list of meals that a user owns (from the User model). We can tell that to our ORM in the User model by adding:

    [InverseProperty("Owner")] // <-- Needs to match the name in Meal
    public List<Meal> OwnedMeals { get; set; }

Now I can do something like:

    int GetTotalMealCalories(User someUser) {
        return someUser.OwnedMeals.Sum(meal => meal.Calories);
    }

## Many-to-Many relationship

Having proteins is nice, but the way things are setup, each meal will only have one protein. What if I'm making a Pizza-Builder that needs to add a bunch of toppings to a single pizza?

Again, we'll need a Topping table (similar to the Protein table from before) that the user can populate by creating their own topping types:

| Id |   Title   | CaloriesPerOunce |
|----|-----------|------------------|
| 1  | Olives    | 35               |
| 2  | Mushroom  | 50               |
| 3  | Pineapple | 20               |
| 4  | Chicken   | 70               |
| 5  | Pepperoni | 9999999          |

But how can we link this up with the Pizza table? If I have a `ToppingId` column on Pizza, then I'll only be able to hold one topping! How can we accomplish a design where a pizza has a bunch of toppings?

**We'll actually need one more table to act as a linker!**

So if my Pizza looks like (where `OwnerId` is a foreign key to the user who created the pizza):

| Id |   Title         | Description                  | OwnerId         |
|----|-----------------|------------------------------|-----------------|
| 1  | Basic Cheese    | Really just cheese honestly  | 4               |
| 2  | Ultra Chicken   | Did someone say chicken?     | 3               |
| 3  | EVERYTHING      | GIVE IT ALL TO ME            | 3               |
| 4  | Best Pizza Ever | The only way to go tbh       | 1               |

Then I could link Pizza and Topping with a PizzaTopping table:

| Id  | ToppingId | PizzaId |
|-----|-----------|---------|
| 1   | 4         | 2       |
| 2   | 4         | 3       |
| 3   | 2         | 3       |
| 4   | 4         | 2       |
| 5   | 4         | 4       |
| 6   | 3         | 4       |
| 7   | 1         | 3       |
| 8   | 5         | 3       |
| 9   | 4         | 2       |
| 10  | 3         | 3       |
| 11  | 2         | 4       |
| 12  | 4         | 2       |

What on earth does that mean? It's just a bunch of numbers!

 Since each topping can be on multiple pizzas, and each pizza can have multiple toppings, we need this extra table to tell us which toppings are where and which pizzas have what. So for example, if I wanted to know what the "Ultra Chicken" pizza has on it, I'd look at its Id, which is `2`, and then I'd look for all rows in the PizzaTopping table with a `PizzaId` of `2`. Doing that, I see that "Ultra Chicken" pizza has `ToppingId`'s {4, 4, 4, 4} on it. If I look at what that Id corresponds to in the Topping table I see that..."Ultra Chicken" pizza is just the "Chicken" topping 4 times 🤦‍♂️.

Even though this looks pretty hard to read in your database, on the C# side, we can use your ORM to make things easy as pizza pie!

The linker model here will look like:

    using System;
    using System.Collections.Generic;
    using System.ComponentModel.DataAnnotations;
    using System.ComponentModel.DataAnnotations.Schema;
    using System.Linq;
    using System.Threading.Tasks;

    namespace capstone.Models
    {
        public class PizzaTopping
        {
            [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
            [Key]
            public int Id { get; set; }
            
            public int ToppingId { get; set; }
            
            [ForeignKey("ToppingId")]
            public Topping topping { get; set; }
            
            public int PizzaId { get; set; }
            
            [ForeignKey("PizzaId")]
            public Pizza pizza { get; set; }
        }
    }

And then on the Pizza side:

    using System;
    using System.Collections.Generic;
    using System.ComponentModel.DataAnnotations;
    using System.ComponentModel.DataAnnotations.Schema;
    using System.Linq;
    using System.Threading.Tasks;

    namespace capstone.Models
    {
        public class Exercise
        {
            [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
            [Key]
            public int id { get; set; }
            
            public string Title { get; set; }
            public string Description { get; set; }
            
            [InverseProperty("pizza")]
            public List<PizzaTopping> PizzaToppings { get; set; }
        }
    }

Now if I wanted to get the total calories on my pizza, I could do something like:

    public int GetTotalCalories(Pizza p) {
        // This is assuming that each PizzaTopping is one ounce of the topping
        return p.PizzaToppings.Sum(pt => pt.Topping.CaloriesPerOunce);
    }

 ## NOTE:

When fetching data from your db context, any *related* data (`InverseProperties`  or foreign key objects) **will not** be automatically loaded! Your database is lazy, it'll only fetch what it thinks it needs to. So if you're trying to access some related data and you find it's `null`, you may be forgetting to remind the database to fetch them!

For example, if you want to write a query to get a pizza and you want to access its ingredients too, you'll need to manually tell the database to fetch those ingredients!

From the [Microsoft docs on working with data](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/work-with-data-in-asp-net-core-apps):

> When EF Core retrieves entities, it populates all of the properties that are stored directly with that entity in the database. Navigation properties, such as lists of related entities, are not populated and may have their value set to null. This ensures EF Core is not fetching more data than is needed, which is especially important for web applications, which must quickly process requests and return responses in an efficient manner. To include relationships with an entity using _eager loading_, you specify the property using the Include extension method on the query, as shown:

    // .Include requires using Microsoft.EntityFrameworkCore 
    var brandsWithItems = await _context.CatalogBrands 
                                        .Include(b => b.Items) 
                                        .ToListAsync();

In the case of Meal from the first example, if you want to also grab the protein from the db, you'll need to do:

    var myMeal = await _context.Meals
                                .Where(meal => meal.Id == myMealId)
                                .Include(meal => meal.ProteinItem)
                                .ToListAsync();

With Pizza, if I want to get all of the toppings, things are a little different, since I need to grab `PizzaToppings` and then on each PizzaTopping I need to grab its Topping. So that'll look like:

    var myPizza = await _context.Pizzas
                                .Where(pizza => pizza.Id == myPizzaId)
                                .Include(pizza => pizza.PizzaToppings)
                                    .ThenInclude(pizzaTopping => pizzaTopping.topping)
                                .ToListAsync();

And now that both the pizza and all of its ingredients are fully loaded into memory, I can pass this `myPizza` into the `GetTotalCalories` function above.

See also the [Microsoft docs on "related data"](https://docs.microsoft.com/en-us/ef/core/querying/related-data).
