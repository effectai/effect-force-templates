<p align="center"><img src="https://effect.network/img/logo/logo.png" width="400px"></p>

<!-- <p align="center"><img src="https://effect.network/img/logo/logo_icon.png" width="96px"></p> -->

<h1 align="center">Effect Force Templates</h1>

Repository for task templates to use on the Effect Force micro tasking platform.

## Templates Content

1. [Image Classifier](./templates/Image_Classifier/)
2. [Tweet Sentiment Analysis](./templates/TweetSentiment/)
3. [Image Viewer](./templates/ImageViewer/)

## Default Templates

In this repo, you will find some example templates that can be used in order to guide you on building campaigns on Effect Force.
It is important that the design of the tasks are clear and easy to use, as it enables Workers to efficiently complete the work and provide better quality. These templates are a good starting point and serve as an example for you to be able to design tasks that Workers will enjoy working on.

Templates are made up of generic HTML tags. Within which you can add your own content.
So the templates can range from any kind of content, from simple images to complex video or audio content with transcriptions capabilities. 
Ideas can range from map annotations, pixel annotations, bounding box annotations, sentiment analysis, image classification surveys, and more!


## Parameter Substitution

The Generic Templates also allow Requesters to use placeholders in their templates. These placeholders are replaced with values from a list of tasks where each row contains all the values for the parameters of a task. The column names of the list with tasks must match the parameter names used in the template. Requesters can use this parameter substitution capability to create a large number of tasks that all share a common layout and design (template).

For example, you want to create a task that asks Workers to provide keywords for images. You create the template for that task for a single image once, and you use a placeholder `${image_url}` for the source of the image:

```html
<p>Provide keywords for the following image:</p>
<img src="${image_url}" />

<input type="text" name="keywords" />
```

This template can be used with a list of placeholders for each task:

```
"image_url"
"https://force.effect.ai/image1.png"
"https://force.effect.ai/image2.png"
"https://force.effect.ai/image3.png"
```

This will result in three tasks that each contain a different picture but share a common template. A single task will look like this on the Effect Force platform:

<p align="center"><img src="example.png"></p>


## Preview Tool
<!-- TODO -->
<!-- Visit the [preview tool]() to modify your template to your likings. -->

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

## Authors

* **Sjoerd Dijkstra** - *Initial work / Templates* - [GitHub](https://github.com/sjoerd-dijkstra)
* **Jesse Eisses** - *Initial work / Templates* - [GitHub](https://github.com/jeisses)
* **Laurens Verspeek** - *Initial work / Templates* - [GitHub](https://github.com/laurensV)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
