---
layout: post
title: IOS开发--自定义TableView
date: 2015-10-14
category: 技术
tags: 实习工作
keywords: ios
description: tableView进阶
---
# ios--自定义TableView
一般情况来说，我们要对TableView做自定义的话。可以采用两种基本方法：

1.在创建TableViewCell时，通过程序，向UITableViewCell添加子视图。

2.从nib文件中，加载一组子视图。

简单的来说，就是分为代码自定义方式，和nib文件自定义方式。今天我所说的，是通过代码的方式来添加子视图，完成我们的自定义TableView.

我们要自定义的tableview,效果如下图：

![view](http://7xiym9.com1.z0.glb.clouddn.com/tableview.png)

## 创建TableView
1.双击main.storyboard.向主视图中添加一个Table View.

2.选中Table View,按下option+command+6打开关联检查器。
将dataSource和delegate旁边的小圆圈，拖动到View Controller的图标上。完成关联。

3.保持table view的选中状态，按下option+command+4打开属性检查器。滚动到View分区中，在Tag文本框中输入“1”.

如果我们为视图指定了唯一的标记值，在代码中获取视图时，可以使用到它。

##创建UITableViewCell子类

1.创建TestCell.h和TestCell.m文件

2.选中TestCell.h

添加代码如下：

	#import <Foundation/Foundation.h>
	#import <UIKit/UIKit.h>

	//自定义Cell一定要记得，继承UITableViewCell
	//类TestCell的属性声明
	@interface TestCell : UITableViewCell
	@property (copy,nonatomic) NSString *name;
	@property (copy,nonatomic) NSString *color;
	@end

正如注释所说，我们要创建一个新的UITableViewCell子类，那么就必须继承UITableViewCell。
而添加的两个属性，将在接下来用到。

3.单击TestCell.m

添加代码如下：

	#import <Foundation/Foundation.h>
	#import "TestCell.h"
    
    //声明
	@interface TestCell()

	@property (strong,nonatomic) UILabel *nameLabel;
	@property (strong,nonatomic) UILabel *colorLabel;

	@end



	//类TestCell的实现
	@implementation TestCell


	//initWithStyle:reuseIdentifier:方法
	-(id)initWithStyle:(UITableViewCellStyle)style 
	reuseIdentifier:(NSString 	*)reuseIdentifier
	{

    		self=[super initWithStyle:style
                reuseIdentifier:reuseIdentifier];
    if (self) {
        //初始化代码
        
        //定义name Label
        CGRect nameLabelRect=CGRectMake(0, 5, 70, 15);//x,y,width,height
        UILabel *nameMarker=[[UILabel alloc]
                             initWithFrame:nameLabelRect];
        nameMarker.textAlignment=NSTextAlignmentRight;
        nameMarker.text=@"Name:";
        nameMarker.font=[UIFont boldSystemFontOfSize:12];
        [self.contentView addSubview:nameMarker];
        
        //定义color Label
        CGRect colorLabelRect=CGRectMake(0, 26, 70, 15);
        UILabel *colorMarker=[[UILabel alloc]
                             initWithFrame:colorLabelRect];
        colorMarker.textAlignment=NSTextAlignmentRight;
        colorMarker.text=@"Color:";
        colorMarker.font=[UIFont boldSystemFontOfSize:12];
        [self.contentView addSubview:colorMarker];
        
        
        //name Label右边要显示的value Label
        CGRect nameValueRect=CGRectMake(80, 5, 200, 15);
        _nameLabel=[[UILabel alloc]
                    initWithFrame:nameValueRect];
        [self.contentView addSubview:_nameLabel];
        
        //color Label右边要显示的value Label
        CGRect colorValueRect=CGRectMake(80, 25, 200, 15);
        _colorLabel=[[UILabel alloc]
                     initWithFrame:colorValueRect];
        [self.contentView addSubview:_colorLabel];
        
        
    }
    		return self;

	}

	-(void)setName:(NSString *)n
	{
    if(![n isEqualToString:_name]){//_name为h文件中定义的name
        _name=[n copy];
        self.nameLabel.text=_name;
    }
	}
	-(void)setColor:(NSString *)c
	{
    if (![c isEqualToString:_color]) {
        _color=[c copy];
        self.colorLabel.text=_color;
    }
	}
	@end
	
	
上面的代码，主要是在TestCell中，定义了四个要显示的Label.以及setName和setColor的方法用于设置两个label的text的值。

## 编写控制器代码
1.单击我们的ViewController.h文件。

添加代码如下：

	#import <UIKit/UIKit.h>

	@interface ViewController : 			
	UIViewController<UITableViewDataSource,UITableViewDelegate>
	@end
	
	
上面代码的作用，是让类遵循两个协议。即：UITableViewDataSource,UITableViewDelegate。类只用遵循这两个协议，才能成为table view的委托和数据源。

2.单击ViewController.m文件。

添加代码如下：

	#import "ViewController.h"
	#import "TestCell.h"

	 static NSString *CellTabIdentifier=@"CellTableIdentifier";
	
	
	@interface ViewController ()
	
	@property (copy,nonatomic) NSArray *computers;
	
	@end



	@implementation ViewController

	- (void)viewDidLoad {
       [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    //视图加载完成之后，通常是从nib文件加载，做额外的设置
    self.computers=@[@{@"Name":@"Macbook",@"Color":@"Silver"},@{@"Name":@"Air",@"Color":@"Silver"},@{@"Name":@"Dellbook",@"Color":@"Silver"},@{@"Name":@"AirMini",@"Color":@"Black"}];
    
    //storyboard中指定的tableview,vieWithTag=1
       UITableView *tableView=(id)[self.view viewWithTag:1];
    //注册表单元类，以供后期反复使用。
      [tableView registerClass:[TestCell class] 
      			forCellReuseIdentifier:CellTabIdentifier];
    
      UIEdgeInsets contentInset=tableView.contentInset;
      contentInset.top=20;
      [tableView setContentInset:contentInset];
	}

	- (void)didReceiveMemoryWarning {
    		[super didReceiveMemoryWarning];
    		// Dispose of any resources that can be recreated.
	}


	//返回的section
	-(NSInteger)tableView:(UITableView *)tableView 
	  numberOfRowsInSection:(NSInteger)section
	{
      return [self.computers count];
	}


	-(UITableViewCell *)tableView:(UITableView *)tableView 
	 cellForRowAtIndexPath:(NSIndexPath *)indexPath
	{
    //定义cell
   
    TestCell *cell=[tableView dequeueReusableCellWithIdentifier:CellTabIdentifier
                    forIndexPath:indexPath];
    
    
    //使用indexPath参数，确定表正在请求单元的哪一行。然后用该行的值为请求的行获取相应的字典。
      NSDictionary *rowData=self.computers[indexPath.row];
    
    //使用所选行中获取的数据来填充cell(单元)
      cell.name=rowData[@"Name"];
      cell.color=rowData[@"Color"];
    
    
      return cell;
		}
		
		
	@end

我们需要多加注意的是viewDidLoad方法，我们在里面注册了TableViewCell子类，即我们之前定义的TestCell.还有tableView:cellForRowAtIndexPath:方法中，我们在里面使用了TestCell，以及对cell中数据的设置。

完成上述步骤之后。command+R运行一下，就可以看到我们要实现的效果了。

本文参考资料：

>《精通IOS开发（第六版）》人民邮电出版社