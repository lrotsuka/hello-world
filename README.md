# hello-world
this is a test

Hi! 

classdef card_thinkspeak_example < matlab.apps.AppBase

    % Properties that correspond to app components
    properties (Access = public)
        UIFigure                matlab.ui.Figure
        MyPlayerNumButtonGroup  matlab.ui.container.ButtonGroup
        Player1Button           matlab.ui.control.RadioButton
        Player2Button           matlab.ui.control.RadioButton
        StatusLabel             matlab.ui.control.Label
        StartButton             matlab.ui.control.Button
        DrawButton              matlab.ui.control.Button
    end

    
    properties (Access = private)
        myPlayerNum % Am I player 1 or player 2?
        otherPlayerNum % The number of the player whose turn is after mine
        channelID % your Channel ID
        writeKey % channel Write API Key
        readKey  % channel Read API Key
        readDelay % wait 5 seconds between ThingSpeak Channel reads
        writeDelay  
        deck   % shuffled deck of cards
        myCard % card that this player most recently drew
        otherPlayerCard % card that the other player most recently drew
    end
    
    methods (Access = private)
        
        function [] = WaitForOtherPlayer(app)
            playerWhoSentMsg = 0;
            
            while playerWhoSentMsg ~= app.otherPlayerNum
                % read a data table from the channel
                dataTable = thingSpeakRead(app.channelID, 'Fields', 1, 'ReadKey', ...
                    app.readKey, 'OutputFormat','Table');
                              
                % access the entry in the table containing the data string
                dataString = string(dataTable.dataString(1));
                
                % to help with debugging
                disp("I am player " + num2str(app.myPlayerNum) ...
                    + " and I just read this data from the ThingSpeak channel:");
                disp(dataString);
                
                % split data string into separate numbers
                dataParts = split(strip(dataString));
                
                % the first number in the string should 
                % always be the number of the player who
                % last wrote the data
                playerWhoSentMsg = str2num(dataParts(1));
                
                % delay before trying to read the online channel again
                pause(app.readDelay);              
            end
            
            % The second and third numbers represent the card 
            % most recently drawn by the other player.
            app.otherPlayerCard.suit = str2num(dataParts(2));
            app.otherPlayerCard.num = str2num(dataParts(3));
            
            % The numbers from the 4th onwards represent 
            % the cards remaining in the deck.
            app.deck = app.MakeDeckFromStringArray(dataParts(4:end));
        end     
        
        function [] = ClearThinkSpeakChannel(app)
            thingSpeakWrite(app.channelID, 'Fields', 1, 'Values', "0", 'WriteKey', app.writeKey);
        end
        
        function [cardName] = GetCardName(app, card)   
            % a card with a suit of 0 means no card or empty card
            if card.suit == 0
                cardName = "";
                return;
            end
            
            suit = card.suit;
            num = card.num;
            
            suffix = "";
            if suit == 1
                suffix = " of diamonds"; 
            elseif suit == 2
                suffix = " of clubs"; 
            elseif suit == 3
                suffix = " of hearts"; 
            else
                suffix = " of spades"; 
            end
            prefix = "";
            if num == 1
                prefix = "Ace" ;
            elseif num == 11
                prefix = "Jack";
            elseif num == 12
                prefix = "Queen";
            elseif num == 13
                prefix = "King";
            else
                prefix = string(num);
            end
            
            cardName = prefix + suffix;         
        end
        
        function [deck] = MakeDeckFromStringArray(app, stringArray)
            deck = [];
            
            for i = 1:2:length(stringArray)
                card.suit = str2num(stringArray(i));
                card.num = str2num(stringArray(i + 1));
                
                deck = [deck, card];
            end
        end
        
        function [deckString] = MakeStringFromDeck(app, deck)
            deckString = "";
            
            for i = 1:length(deck)
                card = deck(i);
                
                suitString = num2str(card.suit);
                numString = num2str(card.num);
                
                deckString = strcat(deckString, suitString, " ", numString, " ");
            end
        end
        
        function [] = UpdateStatus(app, myCardString, otherPlayerCardString)
            p1CardString = "";
            p2CardString = "";
            
            if app.myPlayerNum == 1
                p1CardString = myCardString;
                p2CardString = otherPlayerCardString; 
            else
                p1CardString = otherPlayerCardString;
                p2CardString = myCardString; 
            end
            
            app.StatusLabel.Text = ["Player 1: " + p1CardString ; "Player 2: " + p2CardString];
        end
        
        function [] = SendDataToOtherPlayer(app)
            pause(app.writeDelay);
            dataString = strcat(string(app.myPlayerNum), " ", ...
                string(app.myCard.suit), " ", ...
                string(app.myCard.num), " ");
            
            deckString = app.MakeStringFromDeck(app.deck);
            dataString = strcat(dataString, deckString);
            
            disp("I sent this data to the ThingSpeak channel: ");
            disp(dataString);
            
            thingSpeakWrite(app.channelID, 'Fields', 1, 'Values', dataString ...
                ,'WriteKey', app.writeKey); 
        end
        
        function [] = Shuffle(app)
            % randperm returns a row vector 
            % containing a random permutation of the integers from 1 to n 
            app.deck = app.deck(randperm(length(app.deck)));
        end
        
        function [] = ResetDeck(app)
            app.deck = [];
            for suit = 1:4         
               for num = 1:13
                   card.suit = suit;
                   card.num = num;
                   app.deck = [app.deck, card]; 
               end           
           end
           
           app.Shuffle();
        end
        
        function [drawnCard] = DrawCard(app)
           drawnCard = app.deck(end);
           app.deck(end) = [];
        end
        
        function [] = ClearPlayerCards(app)
            app.myCard.suit = 0;
            app.otherPlayerCard.suit = 0;
        end
    end
    

    % Callbacks that handle component events
    methods (Access = private)

        % Code that executes after component creation
        function startupFcn(app)
            app.myPlayerNum = 1;
            app.otherPlayerNum = 2;
            app.deck = [];
            
             % ASSIGN THE ID OF YOUR OWN CHANNEL HERE
            app.channelID = XXXXXXXXXXX;
            
             % ASSIGN THE WRITE API KEY OF YOUR OWN CHANNEL HERE
            app.writeKey = XXXXXXXXXXX;
            
             % ASSIGN THE READ API KEY OF YOUR OWN CHANNEL HERE
            app.readKey = XXXXXXXXXXX;
            
            app.readDelay = 5;
            app.writeDelay = 1;
            app.ClearThinkSpeakChannel();
            set(app.DrawButton, 'Enable', 'off');
            
            app.myCard.suit = 0;
            app.otherPlayerCard.suit = 0;
        end

        % Selection changed function: MyPlayerNumButtonGroup
        function MyPlayerNumButtonGroupSelectionChanged(app, event)
            selectedButton = app.MyPlayerNumButtonGroup.SelectedObject;
            if selectedButton == app.Player1Button
                app.myPlayerNum = 1;
                app.otherPlayerNum = 2;
            else
                app.myPlayerNum = 2;
                app.otherPlayerNum = 1;
            end
        end

        % Button pushed function: StartButton
        function StartButtonPushed(app, event)
            set(app.Player1Button, 'Enable', 'off');
            set(app.Player2Button, 'Enable', 'off');
            set(app.StartButton, 'Enable', 'off');

            if (app.myPlayerNum == 1)
                set(app.DrawButton, 'Enable', 'on');
                app.ResetDeck();
            else % app.myPlayerNum == 2
                app.WaitForOtherPlayer();
                app.UpdateStatus(app.GetCardName(app.myCard), ...
                                 app.GetCardName(app.otherPlayerCard));
                set(app.DrawButton, 'Enable', 'on');
            end            
        end

        % Button pushed function: DrawButton
        function DrawButtonPushed(app, event)
           app.myCard = app.DrawCard();
           

           app.UpdateStatus(app.GetCardName(app.myCard), ...
                            app.GetCardName(app.otherPlayerCard));
                        
           set(app.DrawButton, 'Enable', 'off');
           
           app.SendDataToOtherPlayer();
          
           % At this point, player 2 is already displaying both cards
           % so the cards variables can be set to blank.         
           if app.myPlayerNum == 2
               app.ClearPlayerCards();
           end
          
           app.WaitForOtherPlayer();
           
           app.UpdateStatus(app.GetCardName(app.myCard), ...
                            app.GetCardName(app.otherPlayerCard));
            
           % At this point, player 1 is already displaying both cards
           % so the cards variables can be set to blank.         
           if app.myPlayerNum == 1
               app.ClearPlayerCards();
           end
           
           set(app.DrawButton, 'Enable', 'on');           
        end
    end

    % Component initialization
    methods (Access = private)

        % Create UIFigure and components
        function createComponents(app)

            % Create UIFigure and hide until all components are created
            app.UIFigure = uifigure('Visible', 'off');
            app.UIFigure.Position = [100 100 640 480];
            app.UIFigure.Name = 'UI Figure';

            % Create MyPlayerNumButtonGroup
            app.MyPlayerNumButtonGroup = uibuttongroup(app.UIFigure);
            app.MyPlayerNumButtonGroup.SelectionChangedFcn = createCallbackFcn(app, @MyPlayerNumButtonGroupSelectionChanged, true);
            app.MyPlayerNumButtonGroup.Position = [248 344 123 62];

            % Create Player1Button
            app.Player1Button = uiradiobutton(app.MyPlayerNumButtonGroup);
            app.Player1Button.Text = 'Player 1';
            app.Player1Button.Position = [11 35 66 22];
            app.Player1Button.Value = true;

            % Create Player2Button
            app.Player2Button = uiradiobutton(app.MyPlayerNumButtonGroup);
            app.Player2Button.Text = 'Player 2';
            app.Player2Button.Position = [11 13 66 22];

            % Create StatusLabel
            app.StatusLabel = uilabel(app.UIFigure);
            app.StatusLabel.FontName = 'Arial Black';
            app.StatusLabel.FontSize = 14;
            app.StatusLabel.Position = [200 158 316 148];
            app.StatusLabel.Text = 'Status';

            % Create StartButton
            app.StartButton = uibutton(app.UIFigure, 'push');
            app.StartButton.ButtonPushedFcn = createCallbackFcn(app, @StartButtonPushed, true);
            app.StartButton.Position = [259 82 100 22];
            app.StartButton.Text = 'Start';

            % Create DrawButton
            app.DrawButton = uibutton(app.UIFigure, 'push');
            app.DrawButton.ButtonPushedFcn = createCallbackFcn(app, @DrawButtonPushed, true);
            app.DrawButton.Position = [64 174 100 118];
            app.DrawButton.Text = 'Draw';

            % Show the figure after all components are created
            app.UIFigure.Visible = 'on';
        end
    end

    % App creation and deletion
    methods (Access = public)

        % Construct app
        function app = card_thinkspeak_example

            % Create UIFigure and components
            createComponents(app)

            % Register the app with App Designer
            registerApp(app, app.UIFigure)

            % Execute the startup function
            runStartupFcn(app, @startupFcn)

            if nargout == 0
                clear app
            end
        end

        % Code that executes before app deletion
        function delete(app)

            % Delete UIFigure when app is deleted
            delete(app.UIFigure)
        end
    end
    hello!!
end
