# The Angular Bundle Optimizer under the hoods

In my last article, I've shown that the [Angular Build Optimizer](https://github.com/angular/devkit/tree/master/packages/angular_devkit/build_optimizer) transforms the emitted JavaScript Code to make tree shaking more efficient. To demonstrate this, I've created a simple scenario that includes two modules of Angular Material without using them. After using the Bundle Optimizer, the CLI/ webpack was able to reduce the bundle size by about the half leveraging tree shaking:

![Reducing the bundle size by about 50 % using tree shaking and the Angular Optimizer](http://i.imgur.com/TyY5HpZ.png) 

If you are wondering how such amazing results are possible, you can find some answers in this article. 

## Tree Shaking and Side Effects

The CLI uses webpack for the build process and to make tree shaking possible, webpack marks exports that are not used and can therefore be safely excluded. In addition to this, a typical webpack configuration uses UglifyJS for removing these exports. Uglify tries to be on the safe side and does not remove any code that could be need at runtime. For instance, when Uglify finds out that some code *could* produce side effects, it keeps it in the bundle. Look at the following (artificial and obvious) example that demonstrates this:

```
(function(exports){ 

	exports.pi = 4; // Let's be generous!

})(...);
```

Unfortunately, when transpiling ES2015+/TypeScript classes down to ES5, the class declaration results in imperative code and UglifyJS as well as other tools cannot make sure that this code isn't producing side effects. A good discussion regarding this can be found [here on GitHub](https://github.com/mishoo/UglifyJS2/issues/1261). That's why the code in question stays in the bundle even though it could be removed.

To assure myself about this fact, I've created a simple npm based Angular Package using the [Jurgen Van de Moere's](https://www.jvandemo.com/) [Yeoman generator for Angular libraries](https://www.npmjs.com/package/generator-angular2-library) as well as a CLI based application that references it. The package's entry point exports an Angular module with an ``UnusedComponent`` that -- as it's name implies -- isn't used by the application. It also exports an ``UnusedClass``. In addition to that, it exports an other ``UnusedClass`` as well as an ``UsedClass`` from the same file (ES Module).  

```
export {UnusedClass} from './unused';
export {OtherUnusedClass, UsedClass} from './partly-used';
export {SampleComponent} from './sample.component';
export {UnusedComponent} from './unused.component';

@NgModule({
  imports: [
    CommonModule
  ],
  declarations: [
    SampleComponent,
    UnusedComponent
  ],
  exports: [
    SampleComponent,
    UnusedComponent
  ]
})
export class SampleModule {
}
```

When building the whole application without the Angular Build Optimizer, none of the unused classes are tree shaken off. One reason for this is that -- as mentioned above -- Uglify cannot make sure that the transpiled classes don't introduce side effects. 

## Marking pure Code Blocks

To compensate for the shown issue, the Angular Build Optimizer marks transpiled class declarations that are not producing side effects with a special ``/*@__PURE__*/`` comment. UglifyJS on the other side respects those comments and removes such code blocks if not referenced. 

Using this technique, we can get rid of unused classes but not of the unused component. The reason for this is, as the next section shows, Angular's module system.

## Removing Angular Decorators

As the ``NgModule``-Decorator defines an Array with its Components, Directives, Pipes and Services, there is always a reference to them when importing the module. This prevents tools from tree shaking unused building blocks off. But we are lucky, because after AOT compilation Angular's decorators are not needed anymore and therefore the Angular Bundle Optimizer removes them all. This leads to code that isn't referencing unneeded stuff anymore and allows to shake it off.

## Static Members

For static members transpiled code is often introducing side effects preventing tree shaking too. For instance, look at the following example taken from the [samples of the Optimizer's Readme on GitHub](https://github.com/angular/devkit/tree/master/packages/angular_devkit/build_optimizer). It shows the transpiled version of a class with a static member:

```
// Taken from the Angular Build Optimizer repo
var Clazz = (function () { function Clazz() { } return Clazz; }());
Clazz.prop = 1;
```

To prevent the introduced side effect, the Angular Build Optimizer rewrites such sources as follows:

```
var Clazz = (function () { function Clazz() { } Clazz.prop = 1; return Clazz; }());
```

## Using the Bundle Optimizer

Because of the three mentioned transformations the Angular Bundle Optimizer performs, the above mentioned unused classes and the unused component can be shaken off by the CLI and webpack. You can try it out yourself: Just download the [example from my GitHub repository](https://github.com/manfredsteyer/angular-build-optimizer-experiment-02) and create a production build with and without the Bundle Optimizer. For the latter build, you need to use the command line option ``-bo``:

```
ng build --prod -bo
```

Please make sure that you have a current CLI version. When writing this, I've used version 1.3.0-rc.0 which was the first one coming with the Optimizer. After this, look into the generated bundles. While the one that has been created without the Optimizer contains all the unused classes, the other one doesn't.
 