# 构建JVM语言 - Enkel

<h2 align="center">【终章】：用Enkel编写真实的应用</h2>

</br>

[原文](http://jakubdziworski.github.io/enkel/2016/05/19/enkel_20_enkel_real_app_cards.html)

</br>

## 源码

这个项目的源码可以从[Github仓库](https://github.com/JakubDziworski/Enkel-JVM-language)中进行克隆。

## 纸牌绘图模拟器

在前面的19篇文章中我已经描述了我的第一个自制的编程语言。如果我不做一些有用的事，那之前的努力岂不是白费了吗？

我想到一个主意，制作一个纸牌绘图模拟器。这个主意是：提供一定数量的玩家，每个玩家分配一定数量的卡片。作为输出，你将获得每个玩家随机纸牌的集合。就像真实游戏中一样，绘画时会从堆栈中移除一张卡片。

### Card类

```groovy
Card {

    string color
    string pattern

    Card(string cardColor,string cardPattern) {
        color = cardColor
        pattern = cardPattern
    }

    string getColor() {
        color
    }

    string getPattern() {
        pattern
    }

    string toString() {
        return "{" + color + "," + pattern + "}"
    }
}
```

这里没有幻想。只有简单的领域对象。它是扑克牌的不可变的形式。

### CardDrawer类

```groovy
CardDrawer {
    start {
        var cards = new List() //creates java.util.ArrayList 
        addNumberedCards(cards) //calling method with 3 arguments (last 2 are default)
        addCardWithAllColors("Ace",cards) 
        addCardWithAllColors("Queen",cards)
        addCardWithAllColors("King",cards)
        addCardWithAllColors("Jack",cards)
        //Calling with named arguments (and in differnet order)
        //The last parameter (cardsPerPlayer) is ommited (it's default value is 5)
        drawCardsForPlayers(playersAmount -> 5,cardsList -> cards) 
    }

    addNumberedCards(List cardsList,int first=2, int last=10) {
        for i from first to last {  //loop from first to last (inclusive)
            var numberString = new java.lang.Integer(i).toString()
            addCardWithAllColors(numberString,cardsList)
        }
    }

    addCardWithAllColors(string pattern,List cardsList) {
        cardsList.add(new Card("Clubs",pattern))
        cardsList.add(new Card("Diamonds",pattern))
        cardsList.add(new Card("Hearts",pattern))
        cardsList.add(new Card("Spades",pattern))
    }

    drawCardsForPlayers(List cardsList,int playersAmount = 3,int cardsPerPlayer = 5) {
        if(cardsList.size() < (playersAmount * cardsPerPlayer)) {
            print "ERROR - Not enough cards" //No exceptions yet :)
            return
        }
        var random = new java.util.Random()
        for i from 1 to playersAmount {
            var playernumberString = new java.lang.Integer(i).toString()
            print "player " + playernumberString  + " is drawing:"
            for j from 1 to cardsPerPlayer {
                var dawnCardIndex = random.nextInt(cardsList.size() - 1)
                var drawedCard = cardsList.remove(dawnCardIndex)
                print "    drawed:" + drawedCard
            }
        }
    }
}
```

### 运行

首先，我们需要编译Card类（因为没有实现多文件编译）。一旦Card类被编译了，CardDrawer就可以在classpath上找到它（只要我们将当前目录添加到classpath上）

```s
java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar: com.kubadziworski.compiler.Compiler EnkelExamples/RealApp/Card.enk
java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar:. com.kubadziworski.compiler.Compiler EnkelExamples/RealApp/CardDrawer.enk

kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java CardDrawer 
player 1 is drawing:
    drawed:{Diamonds,Queen}
    drawed:{Spades,7}
    drawed:{Hearts,Jack}
    drawed:{Spades,4}
    drawed:{Hearts,2}
player 2 is drawing:
    drawed:{Diamonds,4}
    drawed:{Hearts,Ace}
    drawed:{Diamonds,Jack}
    drawed:{Spades,Queen}
    drawed:{Spades,King}
player 3 is drawing:
    drawed:{Diamonds,Ace}
    drawed:{Clubs,2}
    drawed:{Clubs,3}
    drawed:{Spades,8}
    drawed:{Clubs,7}
player 4 is drawing:
    drawed:{Spades,Ace}
    drawed:{Diamonds,3}
    drawed:{Clubs,4}
    drawed:{Clubs,6}
    drawed:{Diamonds,2}
player 5 is drawing:
    drawed:{Hearts,4}
    drawed:{Hearts,Queen}
    drawed:{Hearts,10}
    drawed:{Clubs,Jack}
    drawed:{Diamonds,8}
```

非常好！这证明Enkel虽然在前期阶段，却已经能够创建一些真实的应用了。

## 再见Enkel

我在创建Enkel的过程中度过了非常愉快的时光。写代码是一件事，将整个过程记录下来并让其他人可以理解是另一件富有挑战的事情。

在这个过程中我学到了很多，并且希望其他人可以从这个系列文章中收益。

非常遗憾这是这个系列的最后一篇文章。但是这个项目将继续跟进，所以请继续保持关注。

